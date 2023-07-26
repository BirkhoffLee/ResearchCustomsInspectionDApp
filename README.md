# CustomsInspectionDApp

This is the web UI POC of unpublished paper "Decentralized Face Identification with Hierarchical Navigable Small World on Blockchain" by Hsiang-Hung Lee and Yi-ting Chen.

## Usage

[pnpm](https://pnpm.io/) is recommended to install dependencies.

```bash
$ pnpm install
$ pnpm run dev
```

Most logics and functionalities are implemented in [src/App.vue](src/App.vue).

By default, the app assumes the backend is at `http://127.0.0.1:5000` (defined [here](https://github.com/BirkhoffLee/ResearchCustomsInspectionDApp/blob/main/src/App.vue#L17)).

[Metamask](https://metamask.io/) or a modern web3 wallet browser extension is required for blockchain interaction. We used Ethereum Sepolia Testnet during the development.

You should deploy the [contract](https://github.com/BirkhoffLee/ResearchCustomsInspectionContract) with your own wallet should have r/w access to the contract. Modify [src/App.vue#L18-L19](https://github.com/BirkhoffLee/ResearchCustomsInspectionDApp/blob/main/src/App.vue#L18-L19) accordingly.

A webcam is required for face recognition.

## Related Works

* https://github.com/BirkhoffLee/ResearchCustomsInspectionContract
* https://github.com/BirkhoffLee/ResearchFaceRecognitionBackend

## License
This work is licensed under [Apache License 2.0](LICENSE).
