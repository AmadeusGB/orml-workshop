# Open Runtime Module Library (ORML) Workshop
## 1.Introduction
Sub0 Workshop Link: https://www.crowdcast.io/e/axvfinsv/19

This is a workshop project for learning about blockchain runtime development with [Substrate](https://substrate.dev/),[FRAME](https://substrate.dev/docs/en/knowledgebase/runtime/frame) and the[Open Runtime Module Library](https://github.com/open-web3-stack/open-runtime-module-library). This project implements a simple exchange protocol that is built on top of the ORML [Currencies](https://github.com/open-web3-stack/open-runtime-module-library/tree/master/currencies) and [Tokens](https://github.com/open-web3-stack/open-runtime-module-library/tree/master/tokens) pallets. After completing this workshop, participants should have a better understanding of how to design and implement a FRAME pallet.
## 2.Basic requirements before starting
You should have completed at least the following three steps before attempting this tutorial:

- [Substrate Node template engineering environment configuration](https://github.com/AmadeusGB/orml-workshop/blob/main/template-setup.md)
- [Rust local environment configuration](https://github.com/AmadeusGB/orml-workshop/blob/main/rust-setup.md)  

Before attempting this tutorial, you should be familiar with the concepts listed below:
- Launching a Substrate chain
- Submitting extrinsics
- Adding, Removing, and configuring pallets in a runtime
- Pallet design  

## 3.引入相关配置
### 3.1 runtime中lib.rs和cargo.toml配置
**cargo.toml**  
Add the following dependencies:
```bash
[dependencies.serde]
features = ['derive']
version = '1.0.119'

[dependencies.orml-currencies]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

[dependencies.orml-tokens]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

[dependencies.orml-traits]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

std = [
    'orml-currencies/std',
    'orml-tokens/std',
    'orml-traits/std',
    'pallet-exchange/std',
]
```
**lib.rs**
Add the following implement:
```bash
use codec::{Decode, Encode};
use pallet_grandpa::{
	fg_primitives, AuthorityId as GrandpaId, AuthorityList as GrandpaAuthorityList,
};
#[cfg(feature = "std")]
use serde::{Deserialize, Serialize};
use sp_api::impl_runtime_apis;
use sp_consensus_aura::sr25519::AuthorityId as AuraId;
use sp_core::{crypto::KeyTypeId, OpaqueMetadata};
use sp_runtime::traits::{
	AccountIdLookup, BlakeTwo256, Block as BlockT, IdentifyAccount, NumberFor, Verify,
};
use sp_runtime::{
	create_runtime_str, generic, impl_opaque_keys,
	traits::Zero,
	transaction_validity::{TransactionSource, TransactionValidity},
	ApplyExtrinsicResult, MultiSignature, RuntimeDebug,
};
use sp_std::prelude::*;
#[cfg(feature = "std")]
use sp_version::NativeVersion;
use sp_version::RuntimeVersion;

use orml_currencies::BasicCurrencyAdapter;
use orml_traits::parameter_type_with_key;

// A few exports that help ease life for downstream crates.
pub use frame_support::{
	construct_runtime, parameter_types,
	traits::{KeyOwnerProofSystem, Randomness, StorageInfo},
	weights::{
		constants::{BlockExecutionWeight, ExtrinsicBaseWeight, RocksDbWeight, WEIGHT_PER_SECOND},
		IdentityFee, Weight,
	},
	StorageValue,
};
pub use pallet_balances::Call as BalancesCall;
pub use pallet_timestamp::Call as TimestampCall;
use pallet_transaction_payment::CurrencyAdapter;
#[cfg(any(feature = "std", test))]
pub use sp_runtime::BuildStorage;
pub use sp_runtime::{Perbill, Permill};

/// Import the template pallet.
pub use pallet_exchange;

/// An index to a block.
pub type BlockNumber = u32;

/// Alias to 512-bit hash when used in the context of a transaction signature on the chain.
pub type Signature = MultiSignature;

/// Some way of identifying an account on the chain. We intentionally make it equivalent
/// to the public key of our transaction signing scheme.
pub type AccountId = <<Signature as Verify>::Signer as IdentifyAccount>::AccountId;

pub type AccountIndex = u32;

/// Balance of an account.
pub type Balance = u128;

/// Index of a transaction in the chain.
pub type Index = u32;

/// A hash of some data used by the chain.
pub type Hash = sp_core::H256;

/// Digest item type.
pub type DigestItem = generic::DigestItem<Hash>;

#[derive(Encode, Decode, Eq, PartialEq, Copy, Clone, RuntimeDebug, PartialOrd, Ord)]
#[cfg_attr(feature = "std", derive(Serialize, Deserialize))]
pub enum CurrencyId {
	Native,
	DOT,
	KSM,
	BTC,
}

pub type Amount = i128;

impl orml_tokens::Config for Runtime {
	type Event = Event;
	type Balance = Balance;
	type Amount = Amount;
	type CurrencyId = CurrencyId;
	type WeightInfo = ();
	type ExistentialDeposits = ExistentialDeposits;
	type OnDust = ();
}

parameter_type_with_key! {
	pub ExistentialDeposits: |_currency_id: CurrencyId| -> Balance {
		Zero::zero()
	};
}

parameter_types! {
	pub const GetNativeCurrencyId: CurrencyId = CurrencyId::Native;
}

impl orml_currencies::Config for Runtime {
	type Event = Event;
	type MultiCurrency = Tokens;
	type NativeCurrency = BasicCurrencyAdapter<Runtime, Balances, Amount, BlockNumber>;
	type GetNativeCurrencyId = GetNativeCurrencyId;
	type WeightInfo = ();
}

construct_runtime!(
	pub enum Runtime where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		Currencies: orml_currencies::{Pallet, Call, Event<T>},
		Tokens: orml_tokens::{Pallet, Storage, Event<T>, Config<T>},
	}
);
```
### 3.2 Chain_spec.rs and cargo.toml configuration in node
**chain_spec.rs**
Add the following implement:
```bash
use sp_core::{Pair, Public, sr25519};
use node_template_runtime::{
	AccountId, AuraConfig, BalancesConfig, GenesisConfig, GrandpaConfig, TokensConfig,
	SudoConfig, SystemConfig, WASM_BINARY, Signature, CurrencyId,
	NftConfig, items
};
use sp_consensus_aura::sr25519::AuthorityId as AuraId;
use sp_finality_grandpa::AuthorityId as GrandpaId;
use sp_runtime::traits::{Verify, IdentifyAccount};
use sc_service::ChainType;

fn testnet_genesis(
	wasm_binary: &[u8],
	initial_authorities: Vec<(AuraId, GrandpaId)>,
	root_key: AccountId,
	endowed_accounts: Vec<AccountId>,
	_enable_println: bool,
) -> GenesisConfig {
	GenesisConfig {
		orml_tokens: TokensConfig {
			// Prefund with KSM
			endowed_accounts: endowed_accounts.iter().cloned().map(|acc| (acc, CurrencyId::KSM, 1 << 50)).collect(),
		}
	}
}
```

## 4.ORML example: order book use case
Use the ORML module to manually place an order for a trading order. After user A submits a trading pair order, user B can query the order through the front end. If B thinks the price is appropriate, he can initiate a transaction order to complete the transaction.
### 4.1 Configuration dependency
**cargo.toml**  
Add the following dependencies:
```bash
[dependencies.orml-currencies]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

[dependencies.orml-tokens]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

[dependencies.orml-traits]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

[dependencies.orml-utilities]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

std = [
    'orml-currencies/std',
	'orml-tokens/std',
	'orml-traits/std',
	'orml-utilities/std',
	'pallet-balances/std',
]
```
### 4.2 Storage design
**Types**:The order book consists of the name and price of Token A, the name and price of Token B, and the user who submitted the order.
```bash
struct Order<CurrencyId, Balance, AccountId> {
	pub base_currency_id: CurrencyId,
	pub base_amount: Balance,
	pub target_currency_id: CurrencyId,
	pub target_amount: Balance,
	pub owner: AccountId,
}

type BalanceOf<T> =
	<<T as Config>::Currency as MultiCurrency<<T as frame_system::Config>::AccountId>>::Balance;
type CurrencyIdOf<T> =
	<<T as Config>::Currency as MultiCurrency<<T as frame_system::Config>::AccountId>>::CurrencyId;
```
**Storages**:Orders is responsible for recording the order information corresponding to the order ID; NextOrderId is responsible for recording the total number of orders to facilitate pagination.
```bash
Orders：map T::OrderId  => OrderOf<T>>
NextOrderId：T::OrderId

type OrderOf<T> = 
    Order<CurrencyIdOf<T>, BalanceOf<T>, <T as frame_system::Config>::AccountId>;
```
### 4.3 metadata
The custom type in the storage design needs to be manually added in the Polkadot.js setting-developer to be able to operate normally：
```bash
{
    "CurrencyId": {
        "_enum": [
            "Native",
            "DOT",
            "KSM",
            "BTC"
        ]
    },
    "CurrencyIdOf": "CurrencyId",
    "Amount": "i128",
    "AmountOf": "Amount",
    "Order": {
        "base_currency_id": "CurrencyId",
        "base_amount": "Compact<Balance>",
        "target_currency_id": "CurrencyId",
        "target_amount": "Compact<Balance>",
        "owner": "AccountId"
    },
    "OrderOf": "Order",
    "OrderId": "u32"
}
```
### 4.4 functional module
**calls**:submit_order is used to submit a user order; take_order is used to obtain the details of an order under the user address; cancel_order is used to cancel an order under the user address, that is, cancel a pending order.
```bash
pub fn submit_order(
	origin: OriginFor<T>,
	base_currency_id: CurrencyIdOf<T>,
	base_amount: BalanceOf<T>,
	target_currency_id: CurrencyIdOf<T>,
	target_amount: BalanceOf<T>,
)
		
pub fn take_order(
	origin: OriginFor<T>,
	order_id: T::OrderId,
)

pub fn cancel_order(
	origin: OriginFor<T>,
	order_id: T::OrderId,
)
```
**Events**:OrderCreated is used to issue an order creation success event; OrderTaken is used to return order details; OrderCancelled is used to notify users to cancel pending orders.
```bash
OrderCreated(T::OrderId, OrderOf<T>),
OrderTaken(T::AccountId, T::OrderId, OrderOf<T>),
OrderCancelled(T::OrderId),
```
## 5.扩展模块简介
### 5.1 orml-aution
Auction module provides a way to open auction and place bids on-chain. You can open an auction by specifying a start: BlockNumber and/or an end: BlockNumber, and when the auction becomes active enabling anyone to place a bid at a higher price. Trait AuctionHandler is been used to validate the bid and when the auction ends AuctionHandle::on_auction_ended(id, bid) gets called.
**5.1.1 configuration dependencies**
**Runtime**:
```bash
cargo.toml:
[dependencies.orml-aution]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

lib.rs:
impl orml_aution::Config for Runtime ...
```

**5.1.2 interface**
Interfaces available for calling between modules:

- new_auction(start, end) - create an auction
- remove_auction(id) - remove an auction
- update_auction(id, info) - update an auction
- auction_info(id) - return an auction info
- bid(id, value) - Bid an auction

### 5.2 orml-nft
Use orml-nft pallet to create use cases similar to ERC721
**5.2.1 configuration dependencies**
```bash
cargo.toml:
[dependencies.orml-nft]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

lib.rs:
impl orml_nft::Config for Runtime ...
```
**5.22 interface**
Interfaces available for calling between modules:

- create_class(owner, class_metadata, class_data) - create an NFT class
- destroy_class(owner, class_id) - destroy an NFT class
- mint(owner, class_id, token_metadata, token_data) - create an NFT token
- burn(owner, (class_id, token_id)) - burn an NFT token
- transfer(from, to (class_id, token_id)) - transfer an NFT token
- is_owner(owner, (class_id, token_id)) - check if a certain token is owned by the user.

### 5.3 orml-authority
Authority module to provide features for governance including dispatch method on behalf other accounts and schdule dispatchables.
**5.1.1 configuration dependencies**
```bash
cargo.toml:
[dependencies.orml-authority]
default-features = false
git = 'https://github.com/open-web3-stack/open-runtime-module-library.git'
version = '0.4.1-dev'

lib.rs:
impl orml_authority::Config for Runtime ...
```
**5.1.2 interface**
Interfaces available for calling between modules:

- dispatch_as(as_origin, call) - can dispatch a dispatchable on behalf of other origin.
- schedule_dispatch(when, priority, with_delayed_origin, call) - can schdule a dispatchable to be dispatched at later block.
- fast_track_scheduled_dispatch(initial_origin, task_id, when) - can fast track a scheduled dispatchable.
- delay_scheduled_dispatch(initial_origin, task_id, additional_delay) - can delay a scheduled dispatchable.
- cancel_scheduled_dispatch(initial_origin, task_id) - can cancel a scheduled dispatchable.

