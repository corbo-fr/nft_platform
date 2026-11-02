# nft_platform

`nft_platform` is a Hardhat project by Mathis Bois ([@corbo-fr](https://github.com/corbo-fr)). It is a from-scratch Solidity implementation of [ERC-721](https://eips.ethereum.org/EIPS/eip-721), the non-fungible token standard, together with metadata, per-token URI, burn, enumeration, and contract-URI extensions. The Hardhat config, the test file, and the deployment module are TypeScript.

The repository is a portfolio piece and a work in progress. The contracts compile. They are separate building blocks, and the [implementation status](#implementation-status) section describes behavior that is still unfinished. There is no external mint function and no single collection contract that inherits the extensions together.

## Contracts

Sources live in [`contract/contracts/`](contract/contracts/). The core file is written against the ERC-721 specification, uses an extension layout close to OpenZeppelin, and comments the [MagicalPixelord](https://opensea.io/collection/magicalpixelord) collection as inspiration. `@openzeppelin/contracts` is imported only for `IERC721Receiver`.

### ERC721

[`ERC721.sol`](contract/contracts/ERC721.sol) is an abstract base. It stores a balance per address, an owner per token id, one approved address per token id, and operator approvals per owner.

| Function | Role |
| --- | --- |
| `balanceOf` | Returns the token count for an address. Reverts for the zero address (`InvalidAddress`). |
| `ownerOf` | Returns the address stored for a token id. |
| `transferFrom` | Moves a token. Marked `payable`, matching the ERC-721 interface. |
| `safeTransferFrom` | Two `payable` overloads: one with extra `bytes` data, and one without. |
| `approve`, `getApproved` | Set and read the approved address for one token. `approve` is `payable`. |
| `setApprovalForAll`, `isApprovedForAll` | Set and read an operator flag for the caller. |
| `supportsInterface` | ERC-165. Reports `IERC721` and `IERC165`. |

Internal `_mint`, `_safeMint`, and `_update` change ownership. The other custom errors are `NotAuthorizedOperator`, `TokenDoesNotExist`, `FromIsTo`, `TokenOwnerMissMatch`, and `DestinationNotSafe`.

`balanceOf` reverts for the zero address. `approve` and `setApprovalForAll` revert when the operator is the zero address, and `isApprovedForAll` reverts when either address is zero. Transfers revert when `from` or `to` is the zero address, when `from` and `to` are the same account, when `from` is not the stored owner, and when the caller is not the owner, an approved operator, or the approved address for that token. `approve` allows the owner or an operator. `ownerOf`, transfers, `approve`, `getApproved`, and the metadata `tokenURI` functions are guarded by `tokenExists`; the current condition of that modifier is described under [Implementation status](#implementation-status).

The receiver path calls OpenZeppelin's `IERC721Receiver.onERC721Received` and expects selector `0x150b7a02`. The code-size check comes from [`Address.isContract`](contract/contracts/utils/Address.sol).

### Extensions

| Contract | Kind | What the source adds |
| --- | --- | --- |
| [`ERC721Metadata`](contract/contracts/ERC721Metadata.sol) | abstract | Collection `name`, `symbol`, and a base URI. The constructor reverts with `BaseURIEmpty` when the base URI is empty. `tokenURI` returns `abi.encodePacked(baseURI, tokenId, ".json")`, so the token id is packed as a 32-byte word. |
| [`ERC721URIStorage`](contract/contracts/ERC721URIStorage.sol) | abstract | An optional per-token URI. `tokenURI` returns that string when it is set, and otherwise the base-URI form above. Internal `_setTokenURI` stores the string and emits ERC-4906 `MetadataUpdate`. `supportsInterface` also reports interface id `0x49064906`. |
| [`ERC721Burnable`](contract/contracts/ERC721Burnable.sol) | abstract | Public `burn(uint256)` for a caller who passes the transfer authorization check. The body calls `_update` with `msg.sender` as the previous owner and `address(0)` as the recipient. |
| [`ERC721Enumerable`](contract/contracts/ERC721Enumerable.sol) | concrete | `totalSupply` and `tokenOfOwnerByIndex`. An override of `_update` maintains a per-owner index and a global token array on mint, transfer, and burn. `supportsInterface` also reports `IERC721Enumerable`. `tokenByIndex` is declared with an empty body. This contract has no constructor arguments and does not inherit metadata, burn, URI storage, or contract URI. |
| [`ERC7572`](contract/contracts/ERC7572.sol) | abstract | [ERC-7572](https://eips.ethereum.org/EIPS/eip-7572) `contractURI()` and internal `_updateContractURI(string)`. The stored URI starts empty. |

Each extension is its own contract. `ERC721URIStorage` is the only one that stacks two of these modules (`ERC721` and `ERC721Metadata`).

### Interfaces

[`contract/contracts/interfaces/`](contract/contracts/interfaces/) holds local interfaces: `IERC165`, `IERC721`, `IERC721Metadata`, `IERC721Enumerable`, `IERC4906`, and `IERC7572`. Comments in those files record where `is IERC721` (and similar bases) were removed to avoid circular imports. `IERC721` declares `Transfer`, `Approval`, and `ApprovalForAll`. `IERC4906` declares `MetadataUpdate` and `BatchMetadataUpdate`. `IERC7572` declares `ContractURIUpdated`.

## Implementation status

The modules are still being built out. This is the current behavior of the source:

- `tokenExists` reverts when a token id already has a stored owner. `ownerOf`, both transfer functions, `approve`, `getApproved`, `tokenURI`, and `burn` all use that modifier, so they revert for an existing token.
- The `_safeMint` overload without data calls itself. `_destinationSafety` calls `onERC721Received` when the recipient has no code.
- The three-argument `safeTransferFrom` forwards with an external call to `this.safeTransferFrom`. The inner call sees this contract as `msg.sender`, and the attached ether is left on this contract.
- `_update` writes a new owner only when the recipient is non-zero, so a burn leaves the previous owner in place. It also clears the per-token approval.
- `Transfer`, `Approval`, and `ApprovalForAll` are not emitted. `_updateContractURI` does not emit `ContractURIUpdated`. `BatchMetadataUpdate` is declared on the interface and unused.
- `approve` reverts when the approved address is the zero address, so this function does not clear an approval.
- `tokenOfOwnerByIndex` reverts when the index is greater than `balanceOf(owner)`, so an index equal to the balance is accepted. `tokenByIndex` does not read the global token array.
- `ERC721Metadata` does not add `IERC721Metadata` to `supportsInterface`.

The Hardhat test and the Ignition module are still the sample that Hardhat generates for a `Lock` contract. This repository has no `Lock.sol`. `contract/hardhat.config.ts` sets the compiler version and nothing else: no networks, RPC URLs, or accounts.

## Tech stack

- Solidity compiled with solc 0.8.28 (`contract/hardhat.config.ts`). File pragmas are `^0.8.20`, `^0.8.24`, `^0.8.26`, or `>=0.6.2`. With this config the compiler uses the Paris EVM target and leaves the optimizer off.
- [Hardhat](https://hardhat.org/) 2.26.2 and [@nomicfoundation/hardhat-toolbox](https://hardhat.org/hardhat-runner/plugins/nomicfoundation-hardhat-toolbox) 6.1.0. The toolbox supplies ethers v6, Chai matchers, network helpers, Hardhat Ignition, and TypeChain, and it loads Hardhat Verify, hardhat-gas-reporter, and solidity-coverage. `hardhat.config.ts` passes no custom settings for those plugins. A `npx hardhat test` run prints the gas reporter summary table.
- TypeScript for `hardhat.config.ts`, `test/Lock.ts`, and `ignition/modules/Lock.ts`. `tsconfig.json` targets ES2020 and CommonJS, and turns `strict` on.
- [@openzeppelin/contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) 5.4.0, used for `IERC721Receiver`.
- npm. `contract/package.json` names the package `nft_platform` at version 1.0.0.

## Project structure

```text
.
├── .gitignore                         # ignores .todo
└── contract/
    ├── contracts/
    │   ├── ERC721.sol
    │   ├── ERC721Burnable.sol
    │   ├── ERC721Enumerable.sol
    │   ├── ERC721Metadata.sol
    │   ├── ERC721URIStorage.sol
    │   ├── ERC7572.sol
    │   ├── interfaces/
    │   └── utils/Address.sol
    ├── ignition/modules/Lock.ts       # Hardhat sample module
    ├── test/Lock.ts                   # Hardhat sample test
    ├── hardhat.config.ts
    ├── package.json
    ├── package-lock.json
    ├── tsconfig.json
    └── README.md                      # default Hardhat sample text
```

`contract/README.md` is the sample text from a new Hardhat project. This file is the project README.

Hardhat writes `contract/artifacts/`, `contract/cache/`, and `contract/typechain-types/`. `contract/.gitignore` ignores those directories, plus `node_modules`, `.env`, coverage output, and Ignition deployments for chain id 31337.

## Prerequisites

Node.js and npm. Dependencies are locked in `contract/package-lock.json`. Every command below is run from `contract/`.

## Install

```bash
cd contract
npm install
```

## Compile

```bash
npx hardhat compile
```

The compiler builds the 14 Solidity files with solc 0.8.28 and writes TypeChain typings for ethers v6. It warns that these files have no SPDX license identifier: `ERC721.sol`, `ERC721Metadata.sol`, `ERC7572.sol`, `IERC165.sol`, `IERC721.sol`, and `IERC7572.sol`.

## Test

`npm test` runs the placeholder script in `package.json` and exits with `Error: no test specified`.

The Hardhat suite is:

```bash
npx hardhat test
```

`contract/test/Lock.ts` deploys a `Lock` contract, checks its unlock time, owner, and locked balance, and then checks `withdraw`. All nine tests fail with Hardhat error HH700, `Artifact for contract "Lock" not found`, because `Lock.sol` is not in the project. The suite does not exercise the ERC-721 contracts.

## Deploy

`contract/ignition/modules/Lock.ts` is the only deployment module. It deploys `Lock` with an unlock time, defaulting to 1 January 2030 (`1893456000`), and a locked amount, defaulting to 1 gwei:

```bash
npx hardhat ignition deploy ignition/modules/Lock.ts
```

Hardhat cannot deploy that module in this repository: the artifact for `Lock` is missing. No Ignition module or script deploys `ERC721Enumerable` or the other contracts.

`npx hardhat node` starts Hardhat's local JSON-RPC node. The project has no deployment step that runs against that node.

## License

`contract/package.json` sets the package license to ISC.

These Solidity files carry an OpenZeppelin `SPDX-License-Identifier: MIT` header: `ERC721Burnable.sol`, `ERC721Enumerable.sol`, `ERC721URIStorage.sol`, `IERC721Metadata.sol`, `IERC721Enumerable.sol`, and `IERC4906.sol`. `Address.sol` is marked `SPDX-License-Identifier: UNKNOWN`. The remaining Solidity files have no SPDX identifier.
