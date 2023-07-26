<script setup lang="ts">
import { ref, onMounted } from "vue"
import { ethers } from "ethers"

import { Switch, SwitchGroup, SwitchLabel } from '@headlessui/vue'
import { XCircleIcon } from '@heroicons/vue/20/solid'
import { CheckCircleIcon } from '@heroicons/vue/20/solid'

interface IdentityInfo {
  hash: string
  national_id: string
  name: string
  gender: string
  isRestricted: boolean
}

const apiEndpoint = "http://127.0.0.1:5000"
const contractAddress = '0xe90352a7e5a7315232564D5Bf362Bd02B167469f'
const contractABI = [{"inputs":[{"internalType":"string","name":"hash","type":"string"}],"name":"destroyIdentityByHash","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"}],"name":"getAllDataByHash","outputs":[{"components":[{"internalType":"string","name":"nationalID","type":"string"},{"internalType":"string","name":"name","type":"string"},{"internalType":"bool","name":"isRestrictedPersonnel","type":"bool"},{"internalType":"enum Identity.Gender","name":"gender","type":"uint8"}],"internalType":"struct Identity.IdentityInfo","name":"","type":"tuple"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"}],"name":"getGenderByHash","outputs":[{"internalType":"enum Identity.Gender","name":"","type":"uint8"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getHashList","outputs":[{"internalType":"string[]","name":"","type":"string[]"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"}],"name":"getNameByHash","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"}],"name":"getRestrictionStatusByHash","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"},{"internalType":"string","name":"nationalID","type":"string"},{"internalType":"string","name":"name","type":"string"},{"internalType":"bool","name":"isRestrictedPersonnel","type":"bool"},{"internalType":"enum Identity.Gender","name":"gender","type":"uint8"}],"name":"newIdentity","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"string","name":"hash","type":"string"},{"internalType":"bool","name":"isRestrictedPersonnel","type":"bool"}],"name":"setRestrictionStatusByHash","outputs":[],"stateMutability":"nonpayable","type":"function"}]

const people = ref<Array<IdentityInfo>>([])

const isUpdatingIdentityList = ref(false)

// Status
const statusNationalID = ref('N/A')
const statusName = ref('N/A')
const statusGender = ref('N/A')
const statusIsRestricted = ref(false)
const statusRestrictedText = ref('N/A')

// New Identity
const isSubmitting = ref(false)
const isNewIdentitySubmitSuccessful = ref(false)
const new_national_id = ref('')
const new_name = ref('')
const new_gender = ref('Male')
const new_isRestricted = ref(false)

// Identify
const camera = ref<HTMLVideoElement>()
const canvas = ref<HTMLCanvasElement>()
const isIdentifying = ref(false)
const isIdentifyStatusShown = ref(false)
const isIdentifyStatusError = ref(false)
const isIdentifyStatusSuccess = ref(false)

let web3Provider
let web3Signer: ethers.JsonRpcSigner | null = null

const getCameraImageEncoded = () => {
  const el = canvas.value

  if (!el) {
    throw new Error("canvas is null")
  }

  el.getContext('2d')?.drawImage(camera.value as CanvasImageSource, 0, 0, el.width, el.height)

  return el.toDataURL('image/jpeg')
}

const identifyWebcam = async () => {
  if (isIdentifying.value)
    return

  isIdentifying.value = true
  isIdentifyStatusShown.value = false
  isIdentifyStatusError.value = false
  isIdentifyStatusSuccess.value = false

  try {
    const image = getCameraImageEncoded()

    const response = await fetch(`${apiEndpoint}/identify`, {
      method: "POST",
      cache: "no-cache",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ image }),
    })

    const responseData = await response.json()

    if (!responseData.success)
      throw new Error("Backend were not able to identify the personnel.")

    const IdentityContract = new ethers.Contract(contractAddress, contractABI, web3Signer)
    const data = await IdentityContract.getAllDataByHash(responseData.national_hash.toString())

    const info = identityInfoToObject(responseData.hash, data)
    
    if (info.national_id == '')
      throw new Error("Personnel not found in database")

    statusNationalID.value = info.national_id
    statusName.value = info.name
    statusGender.value = info.gender
    statusIsRestricted.value = info.isRestricted
    statusRestrictedText.value = info.isRestricted ? "Yes" : "No"

    isIdentifyStatusSuccess.value = true
    isIdentifyStatusShown.value = true
  } catch (error) {
    console.error(error)

    isIdentifyStatusError.value = true
    isIdentifyStatusShown.value = true

    statusNationalID.value = "N/A"
    statusName.value = "N/A"
    statusGender.value = "N/A"
    statusRestrictedText.value = "N/A"
  }

  isIdentifying.value = false
}

const setupWebcamFeed = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: {
      width: { ideal: 900 },
      height: { ideal: 506 }
    },
    audio: false
  })

  if (camera.value) {
    camera.value.srcObject = stream
  }
}

const setupWeb3Integration = async () => {
  if (window.ethereum == null) {
    // If MetaMask is not installed, we use the default provider,
    // which is backed by a variety of third-party services (such
    // as INFURA). They do not have private keys installed so are
    // only have read-only access
    alert("MetaMask not installed; app will not work properly")

    // TODO: maybe use Infura RPC
  } else {
    // Connect to the MetaMask EIP-1193 object. This is a standard
    // protocol that allows Ethers access to make all read-only
    // requests through MetaMask.
    web3Provider = new ethers.BrowserProvider(window.ethereum)

    // It also provides an opportunity to request access to write
    // operations, which will be performed by the private key
    // that MetaMask manages for the user.
    web3Signer = await web3Provider.getSigner()
  }
}

const identityInfoToObject = (hash: string, raw: Array<any>): IdentityInfo => ({
  hash,
  national_id: raw[0],
  name: raw[1],
  isRestricted: Boolean(raw[2]),
  gender: (Number(raw[3]) == 0) ? "Male" : "Female",
})

const updateIdentityList = async () => {
  if (isUpdatingIdentityList.value)
    throw new Error('Already updating identityList')

  isUpdatingIdentityList.value = true

  const IdentityContract = new ethers.Contract(contractAddress, contractABI, web3Signer)

  const hashList = await IdentityContract.getHashList()

  let newIdentityList = await Promise.all(hashList.map(async (hash: string) => {
    const data = await IdentityContract.getAllDataByHash(hash)
    return identityInfoToObject(hash, data)
  }))

  // sort by national_id
  people.value = newIdentityList.sort((a, b) => (a.national_id > b.national_id) ? 1 : ((b.national_id > a.national_id) ? -1 : 0))

  isUpdatingIdentityList.value = false
}

const removeIdentity = async (hash: string) => {
  try {
    const response = await fetch(`${apiEndpoint}/identity/${hash}`, {
      method: "DELETE",
      cache: "no-cache",
      headers: {
        "Content-Type": "application/json",
      }
    })

    const responseData = await response.json()

    console.log(responseData)

    if (!responseData.success)
      throw new Error("backend error")

    const IdentityContract = new ethers.Contract(contractAddress, contractABI, web3Signer)

    const result = await IdentityContract.destroyIdentityByHash(hash)

    console.log(result)

    // remove this identity from list so user dont click again
    people.value = people.value.filter(p => p.hash != hash)
  } catch (error) {
    alert(error)
  }
}

const updateRestrictionStatus = async (person: IdentityInfo) => {
  try {
    const IdentityContract = new ethers.Contract(contractAddress, contractABI, web3Signer)

    console.log(person)

    // our event fires before .isRestricted changes
    const result = await IdentityContract.setRestrictionStatusByHash(person.hash, !person.isRestricted)

    console.log(result)
  } catch (error) {
    console.error(error)
    alert(error)
    // revert changes
    person.isRestricted = !person.isRestricted
  }
}

const newIdentity = async () => {
  isSubmitting.value = true
  isNewIdentitySubmitSuccessful.value = false

  if (new_national_id.value == '' || new_name.value == '') {
    isSubmitting.value = false
    return
  }

  const nationalIDList = people.value.map(p => p.national_id)

  if (nationalIDList.includes(new_national_id.value)) {
    alert("Error: Passport Number already exists in record")
    isSubmitting.value = false
    return
  }

  try {
    const image = getCameraImageEncoded()

    const response = await fetch(`${apiEndpoint}/identity/new`, {
      method: "POST",
      cache: "no-cache",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ image }),
    })

    const responseData = await response.json()

    console.log(responseData)

    if (!responseData.success)
      throw new Error("backend error")

    const IdentityContract = new ethers.Contract(contractAddress, contractABI, web3Signer)

    const result = await IdentityContract.newIdentity(
      responseData.national_hash,
      new_national_id.value,
      new_name.value,
      (new_gender.value == "Male") ? 0 : 1,
      BigInt(new_isRestricted.value)
    )

    console.log(result)

    isNewIdentitySubmitSuccessful.value = true
  } catch (error) {
    console.error(error)
    alert(error)
  }

  isSubmitting.value = false

  new_national_id.value = ''
  new_name.value = ''
  new_gender.value = 'Male'
  new_isRestricted.value = false
}

onMounted(async () => {
  await setupWebcamFeed()
  await setupWeb3Integration()

  if (web3Signer === null) {
    console.error("web3Signer is null")
  } else {
    await updateIdentityList()
  }
})

</script>

<template>
  <!-- Webcam, Status, New Identity -->
  <div class="flex space-x-7">
    <!-- Webcam -->
    <div class="w-2/3">
      <div class="divide-y divide-gray-200 overflow-hidden rounded-lg bg-white shadow">
        <!-- Header -->
        <div class="px-4 py-5 sm:px-6">
          <p class="font-semibold">Webcam (Live Feed)</p>
        </div>
        <!-- Webcam Element -->
        <div class="px-4 py-5 sm:p-6">
          <video class="rounded-lg" ref="camera" width="900" height="506" autoplay></video>
          <canvas class="hidden" ref="canvas" width="900" height="506"></canvas>
        </div>
        <!-- Controls -->
        <div class="px-4 py-4 sm:px-6 flex space-x-4">
          <button @click="identifyWebcam" :disabled="isIdentifying" type="button"
            class="rounded-md bg-indigo-500 px-3.5 py-2.5 text-sm font-semibold text-white shadow-sm enabled:hover:bg-indigo-400 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-500">
            <svg v-if="isIdentifying" aria-hidden="true" role="status"
              class="inline w-4 h-4 mr-3 text-gray-200 animate-spin dark:text-gray-600" viewBox="0 0 100 101"
              fill="none" xmlns="http://www.w3.org/2000/svg">
              <path
                d="M100 50.5908C100 78.2051 77.6142 100.591 50 100.591C22.3858 100.591 0 78.2051 0 50.5908C0 22.9766 22.3858 0.59082 50 0.59082C77.6142 0.59082 100 22.9766 100 50.5908ZM9.08144 50.5908C9.08144 73.1895 27.4013 91.5094 50 91.5094C72.5987 91.5094 90.9186 73.1895 90.9186 50.5908C90.9186 27.9921 72.5987 9.67226 50 9.67226C27.4013 9.67226 9.08144 27.9921 9.08144 50.5908Z"
                fill="currentColor" />
              <path
                d="M93.9676 39.0409C96.393 38.4038 97.8624 35.9116 97.0079 33.5539C95.2932 28.8227 92.871 24.3692 89.8167 20.348C85.8452 15.1192 80.8826 10.7238 75.2124 7.41289C69.5422 4.10194 63.2754 1.94025 56.7698 1.05124C51.7666 0.367541 46.6976 0.446843 41.7345 1.27873C39.2613 1.69328 37.813 4.19778 38.4501 6.62326C39.0873 9.04874 41.5694 10.4717 44.0505 10.1071C47.8511 9.54855 51.7191 9.52689 55.5402 10.0491C60.8642 10.7766 65.9928 12.5457 70.6331 15.2552C75.2735 17.9648 79.3347 21.5619 82.5849 25.841C84.9175 28.9121 86.7997 32.2913 88.1811 35.8758C89.083 38.2158 91.5421 39.6781 93.9676 39.0409Z"
                fill="#1C64F2" />
            </svg>
            Identify
          </button>
          <!-- Identification Status -->
          <div v-if="isIdentifyStatusShown" v-show="isIdentifyStatusError" class="rounded-md bg-red-50 px-3.5 py-2.5">
            <div class="flex">
              <div class="flex-shrink-0">
                <XCircleIcon class="h-5 w-5 text-red-400" aria-hidden="true" />
              </div>
              <div class="ml-3">
                <h3 class="text-sm font-medium text-red-800">Personnel not found in database</h3>
              </div>
            </div>
          </div>
          <div v-if="isIdentifyStatusShown" v-show="isIdentifyStatusSuccess"
            class="rounded-md bg-green-50 px-3.5 py-2.5">
            <div class="flex">
              <div class="flex-shrink-0">
                <CheckCircleIcon class="h-5 w-5 text-green-400" aria-hidden="true" />
              </div>
              <div class="ml-3">
                <h3 class="text-sm font-medium text-green-800">Identified personnel</h3>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Status, New Identity -->
    <div class="w-1/3 space-y-7">
      <!-- Status -->
      <div class="divide-y divide-gray-200 overflow-hidden rounded-lg bg-white shadow">
        <!-- Header -->
        <div class="px-4 py-5 sm:px-6">
          <p class="font-semibold">Status</p>
        </div>
        <!-- Details -->
        <div class="px-4 py-5 sm:p-6">
          <div class="flex">
            <div class="w-1/2">
              <p class="text-sm text-gray-500">Passport Number</p>
              <p class="font-semibold">{{ statusNationalID }}</p>
            </div>
            <div class="w-1/2">
              <p class="text-sm text-gray-500">Name</p>
              <p class="font-semibold">{{ statusName }}</p>
            </div>
          </div>
          <div class="flex pt-3">
            <div class="w-1/2">
              <p class="text-sm text-gray-500">Gender</p>
              <p class="font-semibold">{{ statusGender }}</p>
            </div>
            <div class="w-1/2">
              <p class="text-sm text-gray-500">Restricted</p>
              <p class="font-semibold text-red-600" v-if="statusIsRestricted">{{ statusRestrictedText }}</p>
              <p class="font-semibold" v-else>{{ statusRestrictedText }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- New Identity -->
      <form action="POST" class="divide-y divide-gray-200 overflow-hidden rounded-lg bg-white shadow"
        v-on:submit.prevent>
        <!-- Header -->
        <div class="px-4 py-5 sm:px-6">
          <p class="font-semibold">New Identity</p>
        </div>

        <!-- Form -->
        <div class="px-3 py-4 sm:p-5">
          <div class="isolate -space-y-px rounded-md shadow-sm">
            <div
              class="relative rounded-md rounded-b-none px-3 pb-1.5 pt-2.5 ring-1 ring-inset ring-gray-300 focus-within:z-10 focus-within:ring-2 focus-within:ring-indigo-600">
              <label for="passport" class="block text-xs font-medium text-gray-900">Passport Number</label>
              <input required type="text" name="passport" id="passport"
                class="block w-full border-0 p-0 text-gray-900 placeholder:text-gray-400 focus:ring-0 sm:text-sm sm:leading-6"
                placeholder="30003001" v-model="new_national_id" />
            </div>
            <div
              class="relative rounded-md rounded-t-none px-3 pb-1.5 pt-2.5 ring-1 ring-inset ring-gray-300 focus-within:z-10 focus-within:ring-2 focus-within:ring-indigo-600">
              <label for="name" class="block text-xs font-medium text-gray-900">Name</label>
              <input required type="text" name="name" id="name"
                class="block w-full border-0 p-0 text-gray-900 placeholder:text-gray-400 focus:ring-0 sm:text-sm sm:leading-6"
                placeholder="John Doe" v-model="new_name" />
            </div>
          </div>

          <div class="flex mt-4">
            <div class="flex w-1/4">
              <input v-model="new_gender" id="new_male" type="radio" value="Male"
                class="mt-1 ml-1 h-4 w-4 border-gray-300 text-indigo-600 focus:ring-indigo-600">
              <label for="new_male" class="ml-3 block text-sm font-medium leading-6 text-gray-900">Male</label>
            </div>
            <div class="flex w-1/4">
              <input v-model="new_gender" id="new_female" type="radio" value="Female"
                class="mt-1 ml-1 h-4 w-4 border-gray-300 text-indigo-600 focus:ring-indigo-600">
              <label for="new_female" class="ml-3 block text-sm font-medium leading-6 text-gray-900">Female</label>
            </div>
          </div>

          <SwitchGroup as="div" class="flex items-center pt-4">
            <Switch v-model="new_isRestricted"
              :class="[new_isRestricted ? 'bg-indigo-600' : 'bg-gray-200', 'relative inline-flex h-6 w-11 flex-shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none focus:ring-2 focus:ring-indigo-600 focus:ring-offset-2']">
              <span aria-hidden="true"
                :class="[new_isRestricted ? 'translate-x-5' : 'translate-x-0', 'pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow ring-0 transition duration-200 ease-in-out']" />
            </Switch>
            <SwitchLabel as="span" class="ml-3 text-sm">
              <span class="font-medium text-gray-900">Restricted Personnel</span>
            </SwitchLabel>
          </SwitchGroup>
        </div>

        <!-- Controls -->
        <div class="px-4 py-4 sm:px-6 flex space-x-4">
          <button :disabled="isSubmitting" @click="newIdentity" type="submit"
            class="rounded-md bg-indigo-500 px-3.5 py-2.5 text-sm font-semibold text-white shadow-sm enabled:hover:bg-indigo-400 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-500">
            <svg v-if="isSubmitting" aria-hidden="true" role="status"
              class="inline w-4 h-4 mr-3 text-gray-200 animate-spin dark:text-gray-600" viewBox="0 0 100 101"
              fill="none" xmlns="http://www.w3.org/2000/svg">
              <path
                d="M100 50.5908C100 78.2051 77.6142 100.591 50 100.591C22.3858 100.591 0 78.2051 0 50.5908C0 22.9766 22.3858 0.59082 50 0.59082C77.6142 0.59082 100 22.9766 100 50.5908ZM9.08144 50.5908C9.08144 73.1895 27.4013 91.5094 50 91.5094C72.5987 91.5094 90.9186 73.1895 90.9186 50.5908C90.9186 27.9921 72.5987 9.67226 50 9.67226C27.4013 9.67226 9.08144 27.9921 9.08144 50.5908Z"
                fill="currentColor" />
              <path
                d="M93.9676 39.0409C96.393 38.4038 97.8624 35.9116 97.0079 33.5539C95.2932 28.8227 92.871 24.3692 89.8167 20.348C85.8452 15.1192 80.8826 10.7238 75.2124 7.41289C69.5422 4.10194 63.2754 1.94025 56.7698 1.05124C51.7666 0.367541 46.6976 0.446843 41.7345 1.27873C39.2613 1.69328 37.813 4.19778 38.4501 6.62326C39.0873 9.04874 41.5694 10.4717 44.0505 10.1071C47.8511 9.54855 51.7191 9.52689 55.5402 10.0491C60.8642 10.7766 65.9928 12.5457 70.6331 15.2552C75.2735 17.9648 79.3347 21.5619 82.5849 25.841C84.9175 28.9121 86.7997 32.2913 88.1811 35.8758C89.083 38.2158 91.5421 39.6781 93.9676 39.0409Z"
                fill="#1C64F2" />
            </svg>
            Submit
          </button>

          <div v-if="isNewIdentitySubmitSuccessful" class="rounded-md bg-green-50 px-3.5 py-2.5">
            <div class="flex">
              <div class="flex-shrink-0">
                <CheckCircleIcon class="h-5 w-5 text-green-400" aria-hidden="true" />
              </div>
              <div class="ml-3">
                <h3 class="text-sm font-medium text-green-800">Submitted</h3>
              </div>
            </div>
          </div>
        </div>
      </form>
    </div>
  </div>

  <!-- Identity List -->
  <div class="mt-8 flow-root">
    <div class="-mx-4 -my-2 overflow-x-auto sm:-mx-6 lg:-mx-8">
      <div class="inline-block min-w-full py-2 align-middle sm:px-6 lg:px-8">
        <div class="overflow-hidden shadow ring-1 ring-black ring-opacity-5 sm:rounded-lg">
          <table class="min-w-full divide-y divide-gray-300">
            <!-- Header -->
            <thead class="bg-gray-50">
              <tr>
                <th scope="col" class="py-3.5 pl-4 pr-3 text-left text-sm font-semibold text-gray-900 sm:pl-6">Passport
                  Number</th>
                <th scope="col" class="px-3 py-3.5 text-left text-sm font-semibold text-gray-900">Name</th>
                <th scope="col" class="px-3 py-3.5 text-left text-sm font-semibold text-gray-900">Gender</th>
                <th scope="col" class="px-3 py-3.5 text-left text-sm font-semibold text-gray-900">Restricted?</th>
                <th scope="col" class="relative py-3.5 pl-3 pr-4 sm:pr-6">
                  <span class="sr-only">Remove</span>
                </th>
              </tr>
            </thead>
            <!-- Identities -->
            <tbody class="divide-y divide-gray-200 bg-white">
              <tr v-for="person in people" :key="person.national_id">
                <td class="whitespace-nowrap py-4 pl-4 pr-3 text-sm font-medium text-gray-900 sm:pl-6">{{
                person.national_id }}
                </td>
                <td class="whitespace-nowrap px-3 py-4 text-sm text-gray-500">{{ person.name }}</td>
                <td class="whitespace-nowrap px-3 py-4 text-sm text-gray-500">{{ person.gender }}</td>
                <td class="whitespace-nowrap px-3 py-4 text-sm text-red-500 font-semibold">
                  <SwitchGroup as="div" class="items-center">
                    <Switch @click="updateRestrictionStatus(person)" v-model="person.isRestricted"
                      :class="[person.isRestricted ? 'bg-indigo-600' : 'bg-gray-200', 'relative inline-flex h-6 w-11 flex-shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none focus:ring-2 focus:ring-indigo-600 focus:ring-offset-2']">
                      <span aria-hidden="true"
                        :class="[person.isRestricted ? 'translate-x-5' : 'translate-x-0', 'pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow ring-0 transition duration-200 ease-in-out']" />
                    </Switch>
                  </SwitchGroup>
                </td>
                <td class="relative whitespace-nowrap py-4 pl-3 pr-4 text-right text-sm font-medium sm:pr-6">
                  <a href="#" @click.prevent="removeIdentity(person.hash)"
                    class="text-red-600 hover:text-red-900">Remove<span class="sr-only">, {{ person.name
                    }}</span></a>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>

  <button :disabled="isUpdatingIdentityList" @click="updateIdentityList" type="button"
    class="mt-5 rounded-md bg-indigo-500 px-3.5 py-2.5 text-sm font-semibold text-white shadow-sm enabled:hover:bg-indigo-400 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-500">
    <svg v-if="isUpdatingIdentityList" aria-hidden="true" role="status"
      class="inline w-4 h-4 mr-3 text-gray-200 animate-spin dark:text-gray-600" viewBox="0 0 100 101" fill="none"
      xmlns="http://www.w3.org/2000/svg">
      <path
        d="M100 50.5908C100 78.2051 77.6142 100.591 50 100.591C22.3858 100.591 0 78.2051 0 50.5908C0 22.9766 22.3858 0.59082 50 0.59082C77.6142 0.59082 100 22.9766 100 50.5908ZM9.08144 50.5908C9.08144 73.1895 27.4013 91.5094 50 91.5094C72.5987 91.5094 90.9186 73.1895 90.9186 50.5908C90.9186 27.9921 72.5987 9.67226 50 9.67226C27.4013 9.67226 9.08144 27.9921 9.08144 50.5908Z"
        fill="currentColor" />
      <path
        d="M93.9676 39.0409C96.393 38.4038 97.8624 35.9116 97.0079 33.5539C95.2932 28.8227 92.871 24.3692 89.8167 20.348C85.8452 15.1192 80.8826 10.7238 75.2124 7.41289C69.5422 4.10194 63.2754 1.94025 56.7698 1.05124C51.7666 0.367541 46.6976 0.446843 41.7345 1.27873C39.2613 1.69328 37.813 4.19778 38.4501 6.62326C39.0873 9.04874 41.5694 10.4717 44.0505 10.1071C47.8511 9.54855 51.7191 9.52689 55.5402 10.0491C60.8642 10.7766 65.9928 12.5457 70.6331 15.2552C75.2735 17.9648 79.3347 21.5619 82.5849 25.841C84.9175 28.9121 86.7997 32.2913 88.1811 35.8758C89.083 38.2158 91.5421 39.6781 93.9676 39.0409Z"
        fill="#1C64F2" />
    </svg>
    Refresh
  </button>
</template>

<style scoped>

</style>
