# ton docs



## Stake

This section describes how to stake.

### Overview
In **Financial** contract there're op-codes `OP::SIMPLE_TRANSFER` and `OP::STAKE`. They're used to perform staking of TON in the contract, resulting in the creation of a corresponding amount of bmTON jettons in exchange for the staked TON. The current pools of bmTON and TON are taken into account, allowing the calculation of the amount of bmTON to be created dynamically. If the operation is successful, the total supply of bmTON and TON is updated, and the newly minted bmTON jettons are sent to the user's wallet address.

- `OP::SIMPLE_TRANSFER`: Calls the mint function with a `query_id` of `0` and null for the `forward_payload`. It is a simple invocation that doesn’t pass any extra data aside from `msg_value` and `fwd_fee`.
- `OP::STAKE`: Calls the mint function with a `query_id` (provided in `in_msg_body`), a `forward_ton_amount` (specified TON amount to forward) and a `forward_payload` (optional cell for forwarding additional data). These parameters enable more complex interactions, such as sending additional data for use in staking-related functions.

When a user stakes a certain amount of TON, the number of bmTON to be minted is determined based on the existing bmTON and TON pools

The number of bmTON minted is calculated using the formula:
`stake_jetton_amount = jetton_total_supply * stake_ton_amount / ton_total_supply`

Where:
- `jetton_total_supply` is the total supply of bmTON before staking.
- `ton_total_supply` is the total amount of staked TON before the staking operation.
- `stake_ton_amount` is the amount of TON to be staked after subtracting any relevant fees (`int stake_ton_amount = msg_value - get_ton_amount_for_stake(forward_ton_amount, fwd_fee)`).

### TL-B scheme for stake message

```
stake#4253c4d5 query_id:uint64 forward_ton_amount:Coins forward_payload:(Either Cell ^Cell) = InternalMsgBody;
```

### Steps to verify the result of the `OP::STAKE`

To ensure the staking operation was executed correctly, follow these steps:

#### Check the updated bmTON and TON pools:
- Verify that the `jetton_total_supply` has increased by the `calculated stake_jetton_amount`.
- Ensure that the `ton_total_supply` has increased by `stake_ton_amount`.

#### Confirm the transfer of bmTON:
- The minted bmTON should be sent to the user’s bmTON jetton wallet address. Verify that the correct amount was transferred.
- The transaction should use the `OP::INTERNAL_TRANSFER`, and the `query_id` should match the identifier in the staking request.

#### Validate any additional data transfers:
- If the staking operation included forwarding any additional payload (`forward_payload`), ensure that it was correctly forwarded with the transfer.

### Common errors that may occur during processing `OP::STAKE` and `OP::SIMPLE_TRANSFER`

When performing operations in a smart contract, various errors can occur due to incorrect input, insufficient funds, or other issues. To handle these errors effectively, it’s important to understand what kind of errors might arise, how to catch them, and how to determine the underlying issue. Below is a detailed guide on common errors, how to detect them, and troubleshooting strategies.

1. `ERROR::INSUFFICIENT_MSG_VALUE`
    - **Cause**: This error occurs when the provided message value is less than the required amount to perform the operation. For example, the amount of TON sent may not be enough to cover the staking amount after subtracting fees.
    - **How to Detect**: Check if the contract throws this error when the condition `msg_value < required_amount` is true.
    - **Troubleshooting**: Ensure that the amount sent in the transaction is enough to cover the required fees and the intended operation.
    Recalculate the required amount based on the operation and include enough extra funds for transaction fees.

2. `ERROR::NOT_BASECHAIN`
    - **Cause**: This error happens when the message is sent from a non-basechain workchain (i.e., not workchain 0). The contract may be designed to only accept transactions from the main chain.
    - **How to Detect**: The contract checks if the sender’s workchain is not equal to 0 and throws this error if true.
    - **Troubleshooting**: Ensure that the transaction is being sent from the basechain (workchain = 0).
    Verify the sender address and ensure that it matches the expected workchain for the operation.

### Example of sending stake message using pytoniq

So, to create a payload cell you need to provide the following params:
- `OP::STAKE` (`0x4253c4d5`) - The operation code indicating the type of operation. Represented by a 32-bit unsigned integer
- `query_id` - A unique identifier for the request, used for tracking the operation. It is a 64-bit unsigned integer that should be unique for each transaction.
- `forward_ton_amount` - The amount of TON to be forwarded in another message or included in the response. This amount is excluded from the staking calculation.
- `forward_payload` - An optional payload that can be included in the message to be forwarded. This data will be passed along with the forwarding message.

Or you can just send tons attached to the empty message to the **Financial** contract

#### Let's write a simple code that builds payload cell and sends message to Financial
```
from pytoniq import WalletV4R2, LiteBalancer, begin_cell
import asyncio
mnemonics = ["your", "mnemonics", "here"]

async def main():
    provider = LiteBalancer.from_mainnet_config(1)
    await provider.start_up()

    OP_STAKE = 0x4253c4d5

    FINANCIAL_ADDRESS = "EQCSxGZPHqa3TtnODgMan8CEM0jf6HpY-uon_NMeFgjKqkEY"

    stake_msg = begin_cell()\
        .store_uint(OP_STAKE, 32)\
        .store_uint(123, 64)\   # example query_id, should be unique for each request
        .store_coins(int(0.01*1e9))\
    .end_cell()

    ton_to_stake = 100  # example value
    wallet = await WalletV4R2.from_mnemonic(provider=provider, mnemonics=mnemonics)
    await wallet.transfer(destination=FINANCIAL_ADDRESS,
                          amount=int(ton_to_stake*1e9),
                          body=stake_msg)

    await provider.close_all()

asyncio.run(main())
```


## Unstake

This section describes how to unstake.

### Overview
To unstake user needs to send message with `OP::BURN` to its wallet and specify amount of bmTON to burn. The TL-B scheme of burn request is following:

```
burn#595f07bc query_id:uint64 jetton_amount:Coins receiver_address:MsgAddressInt forward_payload:(Maybe ^Cell) = InternalMsgBody;
```

Then message with `OP::BURN_NOTIFICATION` comes from **Jetton Wallet** to **Financial** contract. **Financial** checks that sender is valid and message has enough tons for burn notification and if everything is correct it deploys **Unstake Request** with user address (`receiver_address`) and `withdraw_ton_amount` (`int withdraw_ton_amount = muldiv(ton_total_supply, withdraw_jetton_amount, jetton_total_supply)`). The TL-B scheme of burn notification is following:

```
burn_notification#7bdd97de query_id:uint64 withdraw_jetton_amount:Coins owner_address:MsgAddressInt receiver_address:MsgAddressInt forward_payload:(Maybe ^Cell) = InternalMsgBody;
```

After **Unstake Request** is deployed, anyone can send message to it. Contract checks that unlock timestamp has passed and if everything is correct
it sends message to **Financial** with `OP::UNSTAKE`, ton amount, jetton amount and user address. The TL-B scheme of this message is following:

```
unstake#492ab1b3 index:uint64 owner:MsgAddressInt ton_amount:Coins jetton_amount:Coins forward_payload:(Maybe ^Cell) = InternalMsgBody;
```

Then **Financial** checks that its balance is enough (`balance >= withdraw_ton_amount + msg_value`) and if this condition is correct it sends message with `OP::UNSTAKE_NOTIFICATION` and value equal to `unstake_amount - storage_fee` to user. The TL-B scheme of this message is following:

```
unstake_notification#90c80a07 query_id:uint64 forward_payload:(Maybe ^Cell) = InternalMsgBody;
```

Otherwise, it sends message back to **Unstake Request**. The TL-B scheme of this message is following:

```
return_unstake_request#38633538 unlock_timestamp:uint32 = InternalMsgBody;
```

## Tracking unstake requests

This section describes how to tarck unstake requests.

1. **Monitor Events on the Blockchain**
The unstake request process typically involves sending a transaction to the smart contract that initiates the unstaking. The contract may emit events (messages) related to the unstake request, which can be tracked on the blockchain. Look for events such as:
- `OP::UNSTAKE`: When the **Unstake Request** is initiated, a message with the `OP::UNSTAKE` operation code will be sent. This message will contain details about the unstake request, such as the amount of TON and bmTON to be unstaked, the owner's address, and the index of the request.

2. **Invoke `get_unstake_data()` method in Unstake Request contract.**
By invoking this method you can retrieve the details of an unstake request. When **Unstake Request** sends message to **Financial** with `OP::UNSTAKE` it sets `unlock_timestamp` equal to `0`.


# Smart contracts

## Financial

### Overview
The Financial Smart Contract handles the operations for staking and unstaking. It includes functions for receiving transfers, handling administrative tasks, and managing rewards, commissions, and configuration.

### Storage Variables
- `jetton_total_supply`: Total number of bmTON in circulation.
- `ton_total_supply`: Total TON managed by the contract.
- `commission_total_supply`: Accumulated commission to be distributed.
- `commission_factor`: The percentage factor used to calculate commissions.
- `commission_address`: The address to which commissions are sent.
- `admin_address`: Address of the admin with special permissions.
- `transaction_address`: Address authorized to perform transaction-related operations.
- `content`: Contract content, which can be updated by the admin.
- `jetton_wallet_code`: Code of the bmTON jetton wallet used for state initialization.
- `last_lockup_epoch`: The last epoch when the lockup configuration was refreshed.
- `lockup_supply`: Current supply of locked funds.
- `next_lockup_supply`: Amount of funds that are scheduled to be locked in the next lockup period or epoch within the smart contract.
- `later_lockup_supply`: The amount that is planned to be locked in an even later epoch, providing a way to prepare for long-term commitments.
- `next_unstake_request_index`: Index used for deploying new unstake requests.
- `unstake_request_code`: Code used for unstake request initialization.

#### TL-B scheme of storage
```
_ jetton_total_supply:Coins ton_total_supply:Coins commission_total_supply:Coins commission_factor:uint16 commission_address:MsgAddressInt admin_address:MsgAddressInt transaction_address:MsgAddressInt content:^Cell jetton_wallet_code:^Cell ^[last_lockup_epoch:uint32 lockup_supply:Coins next_lockup_supply:Coins later_lockup_supply:Coins next_unstake_request_index:uint64 unstake_request_code:^Cell] = FinancialStorage
```

### Errors
- `ERROR::INVALID_COMMISSION_FACTOR` (46): Thrown when the commission factor provided is invalid.
- `ERROR::NOT_FROM_ADMIN` (73): Raised if a non-admin tries to perform an admin-only operation.
- `ERROR::INSUFFICIENT_MSG_VALUE` (105): Occurs if the message value is insufficient for the requested action.
- `ERROR::NOT_FROM_UNSTAKE_REQUEST` (76): Thrown if an unstake request originates from an unauthorized sender.

### Functionality

#### 1. Staking and Token Management
- `OP::SIMPLE_TRANSFER`
    - Handles a basic transfer of TON and mints bmTON proportionally.
    - The `mint` function is called with the specified parameters to credit bmTON jettons to the sender. If everything is correct it sends message with the following layout:
        TL-B scheme: 
        ```
        internal_transfer#178d4519 query_id:uint64 jetton_amount:Coins from:MsgAddressInt
                        response_address:MsgAddressInt forward_ton_amount:Coins
                        forward_payload:(Either Cell ^Cell) = InternalMsgBody;
        ```
       - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request. Set to `0`.
       - `jetton_amount`: Amount of transferred jettons.
       - `from`: Address of the sender (transfer initiator).
       - `response_address`: Address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins.
       - `forward_ton_amount`: The amount of nanotons to be sent to the destination address. Set to 0.01 TON (`TON_FOR_STAKE_NOTIFICATION`).
       - `forward_payload`: Optional custom data that should be sent to the destination address. Set to `null()`.
    and increases `jetton_total_supply` by `stake_jetton_amount` (`int stake_jetton_amount = muldiv(jetton_total_supply, stake_ton_amount, ton_total_supply)`)
    and `ton_total_supply` by `stake_ton_amount` (`int stake_ton_amount = msg_value - transfer_ton_amount`).

    **Should be rejected if:**
    1. The sender's workchain ID is not equal to the basechain's workchain ID (`ERROR::NOT_BASECHAIN`).
    2. There is no enough TON (with respect to jetton own storage fee guidelines and operation costs) to process operation, deploy receiver's jetton-wallet, send `forward_ton_amount` and stake (`ERROR::INSUFFICIENT_MSG_VALUE`).

- `OP::STAKE`
    - Manages staking requests, extracting additional details such as `query_id`, `forward_ton_amount`, and `forward_payload` from the incoming message.
    - The `mint` function processes staking, allocating bmTON jettons and updating total supply.
    ##### TL-B scheme: 
    ```
    stake#4253c4d5 query_id:uint64 forward_ton_amount:Coins forward_payload:(Maybe ^Cell) = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
    - `forward_ton_amount`: Variable length unsigned integer specifying the amount of TON to be forwarded.
    - `forward_payload`: Optional payload in the form of a reference cell, containing additional data for the staking operation.
    If everything is correct it sends message with the following layout:
        TL-B scheme: 
        ```
        internal_transfer#178d4519 query_id:uint64 jetton_amount:Coins from:MsgAddressInt
                        response_address:MsgAddressInt forward_ton_amount:Coins
                        forward_payload:(Either Cell ^Cell) = InternalMsgBody;
        ```
       - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
       - `jetton_amount`: Amount of transferred jettons.
       - `from`: Address of the sender (transfer initiator).
       - `response_address`: Address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins.
       - `forward_ton_amount`: The amount of nanotons to be sent to the destination address.
       - `forward_payload`: Optional custom data that should be sent to the destination address.
    and increases `jetton_total_supply` by `stake_jetton_amount` (`int stake_jetton_amount = muldiv(jetton_total_supply, stake_ton_amount, ton_total_supply)`)
    and `ton_total_supply` by `stake_ton_amount` (`int stake_ton_amount = msg_value - transfer_ton_amount`).

    **Should be rejected if:**
    1. The sender's workchain ID is not equal to the basechain's workchain ID (`ERROR::NOT_BASECHAIN`).
    2. There is no enough TON (with respect to jetton own storage fee guidelines and operation costs) to process operation, deploy receiver's jetton-wallet, send `forward_ton_amount` and stake (`ERROR::INSUFFICIENT_MSG_VALUE`).

#### 2. Unstake and Burn Notifications
- `OP::BURN_NOTIFICATION`
    - Handles notifications when bmTON jettons are burned, calculates the corresponding TON amount, and initiates an unstake request if conditions are satisfied.
    ##### TL-B scheme:
    ```
    burn_notification#7bdd97de query_id:uint64 amount:Coins owner_address:MsgAddressInt receiver_address:MsgAddressInt forward_payload:(Maybe ^Cell) = InternalMsgBody
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
    - `amount`: Variable length unsigned integer specifying the amount of jettons burned.
    - `owner_address`: Address of the jetton owner.
    - `receiver_address`: Address to receive the result of the burn operation (if applicable).
    - `forward_payload`: Optional payload provided for further processing.
    - If everything is correct it sends message with the following layout:
        TL-B scheme: 
        ```
        deploy_unstake_request#10a1ce75 query_id:uint64 owner_address:MsgAddressInt withdraw_ton_amount:Coins withdraw_jetton_amount:Coins forward_payload:(Maybe ^Cell) lockup_timestamp:uint32 = InternalMsgBody;
        ```
        - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
        - `owner_address`: Address of the user.
        - `withdraw_ton_amount`: Amount of TON to be unstaked.
        - `withdraw_jetton_amount`: Amount of jettons to be unstaked (currently unused).
        - `forward_payload`: Optional additional payload.
        - `unlock_timestamp`: Time after which funds can be withdrawn. (`int unlock_timestamp = (last_lockup_epoch + 2) * TIME::LOCKUP_PERIOD`)
    and decreases `ton_total_supply` by `withdraw_ton_amount` and `jetton_total_supply` by `withdraw_jetton_amount`. Increases `next_unstake_request_index` by `1` and `later_lockup_supply` by `withdraw_ton_amount`. Lockup config is also updated.

    **Should be rejected if:**
    1. Message is not from a valid jetton-wallet (`ERROR::NOT_FROM_JETTON_WALLET`)
    2. There is no enough TON to process operation (`ERROR::INSUFFICIENT_MSG_VALUE`)

- `OP::UNSTAKE`
    ##### TL-B scheme:
    ```
    unstake#492ab1b3 index:uint64 owner_address:MsgAddressInt amount_ton:Coins amount_jetton:Coins forward_payload:(Maybe ^Cell) = InternalMsgBody;
    ```
    - `index`: A 64-bit unsigned integer that identifies the unstake request.
    - `owner_address`: Address of the owner initiating the unstake.
    - `amount_ton`: Variable length unsigned integer for the amount of TON to be unstaked.
    - `amount_jetton`: Variable length unsigned integer for the amount of jettons associated with the unstake (currently unused).
    - `forward_payload`: Optional payload with additional information for the unstake.
    
    - If the contract's balance is sufficient to cover `withdraw_ton_amount + msg_value` (the total amount to be unstaked plus any additional transaction cost)
    it sends message to user with the following layout:
        TL-B scheme:
        ```
        unstake_notification#90c80a07 query_id:uint64 forward_payload:(Maybe ^Cell) = InternalMsgBody;
        ```
        - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
        - `forward_payload`: Optional additional payload.
    and updates lockup config and decreases `lockup_supply` by `withdraw_ton_amount`.
    - Otherwise, it defers the unstake request by scheduling a retry. Sends message back to the unstake-request contract with the following layout:
        TL-B scheme:
        ```
        return_unstake_request#38633538 lockup_timestamp:uint32 = InternalMsgBody;
        ```
        - `lockup_timestamp`: Specifies the moment when the unstake request can be retried.

    **Should be rejected if:**
    - Message is not from a valid unstake-request contract (`ERROR::NOT_FROM_UNSTAKE_REQUEST`).
        
#### 3. On-chain Requests and Responses
- `OP::PROVIDE_WALLET_ADDRESS`
    - Provides a wallet address for a specific owner, resolving to the basechain if applicable.
    ##### TL-B scheme:
    ###### Request:
    ```
    provide_wallet_address#2c76b973 query_id:uint64 owner_address:MsgAddressInt include_address:Bool = InternalMsgBody;
    ```
    ###### Response:
    ```
    take_wallet_address#d1735400 query_id:uint64 wallet_address:MsgAddressInt included_address:(Maybe MsgAddressInt) = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request, allowing the requester to match responses with their requests.
    - `wallet_address`: The bmTON jetton wallet address calculated based on the provided `owner_address`.
    - `included_address`: Optional field containing the `owner_address` if `include_address?` was set to true.

    **Should be rejected if:**
    - There is no enough TON to process operation (`ERROR::INSUFFICIENT_MSG_VALUE`)

- `OP::PROVIDE_CURRENT_QUOTE`
    - Provides the current quote of pools. It is typically used by external parties to request up-to-date information about the contract’s TON and bmTON balances.
    ##### TL-B scheme:
    ###### Request:
    ```
    provide_current_quote#ad83913f query_id:uint64 custom_payload:(Maybe ^Cell) = InternalMsgBody;
    ```
    ###### Response:
    ```
    take_current_quote#0a420458 query_id:uint64 ton_total_supply:uint128 jetton_total_supply:uint128 custom_payload:(Maybe ^Cell) = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request, allowing the requester to match responses with their requests.
    - `ton_total_supply`: The total amount of TON currently in the pool.
    - `jetton_total_supply`: The total amount of bmTON currently in the pool.
    - `custom_payload`: Optional data from the requester, which is returned in the response for reference or additional processing.

    **Should be rejected if:**
    - There is no enough TON to process operation (`ERROR::INSUFFICIENT_MSG_VALUE`)

- `OP::GET_POOLS`
    -  Allows to request current values of pools managed by the contract. This can be useful for obtaining up-to-date pool sizes for decision-making or reporting purposes.
    ##### TL-B scheme:
    ###### Request:
    ```
    get_pools#2a158bc3 query_id:uint64 = InternalMsgBody;
    ```
    ###### Response:
    ```
    get_pools#2a158bc3 query_id:uint64 jetton_total_supply:Coins ton_total_supply:Coins = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request, allowing the requester to match responses with their requests.
    - `jetton_total_supply`: The current number of bmTON held in the pool.
    - `ton_total_supply`: The current TON balance of the pool.

    **Should be rejected if:**
    - There is no enough TON to process operation (`ERROR::INSUFFICIENT_MSG_VALUE`)

#### Get-methods
- `get_jetton_data`
    Returns core jetton data, including total jetton supply, admin address, content, and jetton wallet code.
    ##### TL-B scheme:
    ```
    get_jetton_data#_ = (jetton_total_supply:Coins mintable:Bool admin_address:MsgAddressInt content:^Cell jetton_wallet_code:^Cell)
    ```
    ##### Returns:
    - `jetton_total_supply`: The total supply of bmTON managed by the contract.
    - `admin_address`: The admin’s wallet address.
    - `content`: The content cell of the contract, which may hold metadata or additional information.
    - `jetton_wallet_code`: The code of the bmTON jetton wallet used in the contract.

- `get_full_data`
    Retrieves comprehensive information about the contract’s state, including bmTON and TON supplies, commission settings, wallet addresses, and configuration data.
    ##### TL-B scheme:
    ```
        get_full_data#_ = (
            jetton_total_supply:Coins
            ton_total_supply:Coins
            commission_total_supply:Coins
            commission_factor:uint16
            commission_address:MsgAddressInt
            admin_address:MsgAddressInt
            transaction_address:MsgAddressInt
            content:^Cell
            jetton_wallet_code:^Cell
            unstake_request_code:^Cell
            last_lockup_epoch:uint32
            lockup_supply:Coins
            next_lockup_supply:Coins
            later_lockup_supply:Coins
            next_unstake_request_index:uint64
        )
    ```
    ##### Returns:
    - `jetton_total_supply`: The total supply of bmTON.
    - `ton_total_supply`: The total supply of TON coins.
    - `commission_total_supply`: The accumulated commission balance held in the contract.
    - `commission_factor`: The factor used to calculate the commission fee.
    - `commission_address`: The wallet address where commissions are sent.
    - `admin_address`: The wallet address of the contract administrator.
    - `transaction_address`: The wallet address used for transaction administration.
    - `content`: The content cell of the contract, often used for metadata or other relevant information.
    - `jetton_wallet_code`: The code of the bmTON jetton wallet utilized in the contract.
    - `unstake_request_code`: The code used for handling unstake requests.
    - `last_lockup_epoch`: The epoch time of the last lockup update.
    - `lockup_supply`: The amount of TON coins currently locked up.
    - `next_lockup_supply`: The amount of TON coins that will be locked up in the next period.
    - `later_lockup_supply`: The amount of TON coins set to be locked up in a later period.
    - `next_unstake_request_index`: The index for the next unstake request.

- `get_wallet_address`
    Returns bmTON jetton wallet address (MsgAddressInt) for this owner address (MsgAddressInt).
    ##### Parameters:
    - `owner_address`: The wallet address of the Jetton token owner.
    ##### Returns:
    The bmTON jetton wallet address corresponding to the given `owner_address`.

- `get_unstake_request_address`
    Returns address of an unstake request (MsgAddressInt) based on the provided index (uint64).
    ##### Parameters:
    - `index`: The index associated with the unstake request.
    ##### Returns:
    The unstake request address corresponding to the given `index`.

- `get_lockup_period`
    ##### Returns:
    The lockup period duration (defined by `TIME::LOCKUP_PERIOD` in seconds).


## Unstake Request

### Overview
This Smart Contract manages individual unstake requests within a decentralized staking system. It handles the process of withdrawing staked funds, ensuring time-locked conditions, and enabling secure fund transfers.

It works in conjunction with a **Financial** contract, which deploys and interacts with the **Unstake Request** contracts.

### Storage Variables
- `index`: A unique identifier for the unstake request.
- `financial_address`: The address of the **Financial** contract handling the funds.
- `owner_address`: The address of the owner making the unstake request.
- `ton_amount`: The amount of TON coins to be unstaked.
- `jetton_amount`: The amount of bmTON to be unstaked.
- `forward_payload`: Optional payload data forwarded with the unstake request.
- `unlock_timestamp`: The timestamp after which the funds can be unstaked.

#### TL-B scheme of storage
```
index:uint64 financial_address:MsgAddressInt owner_address:MsgAddressInt ton_amount:Coins jetton_amount:Coins forward_payload:^Cell unlock_timestamp:uint32 = UnstakeRequestStorage
```

### Errors
- `ERROR::NOT_ALLOWED` (50):
    Indicates that an operation is not allowed because the contract is not in the appropriate state.
- `ERROR::UNLOCK_TIMESTAMP_HAS_NOT_EXPIRED_YET` (51):
    Indicates that the unstaking operation is being attempted before the `unlock_timestamp` has been reached.
- `ERROR::UNKNOWN_OP` (0xffff):
    Indicates that an unknown or unsupported operation (op) code was provided in the message body.

### Functionality
#### 1. Financial Operations
- `OP::DEPLOY_UNSTAKE_REQUEST`
   - Initializes an unstake request with the owner’s address, TON and bmTON amounts, and unlock timestamp.
   - Validates the sender is the **Financial** contract.
    ##### TL-B scheme:
    ```
    deploy_unstake_request#10a1ce75 owner_address:MsgAddressInt ton_amount:Coins jetton_amount:Coins forward_payload:(Maybe ^Cell) unlock_timestamp:uint32 = InternalMsgBody;
    ```
    - `owner_address`: Address of the unstake requester.
    - `ton_amount`: TON amount to be unstaked.
    - `jetton_amount`: bmTON amount to be unstaked.
    - `forward_payload`: Optional additional payload.
    - `unlock_timestamp`: Time after which funds can be withdrawn.

- `OP::RETURN_UNSTAKE_REQUEST`
   - Cancels or modifies an unstake request.
   - Validates the sender is the **Financial** contract.
   - Updates the `unlock_timestamp` to delay the unstake process or return the request.
    ##### TL-B scheme:
    ```
    return_unstake_request#38633538 unlock_timestamp:uint32 = InternalMsgBody;
    ```
    - `unlock_timestamp`: New unlock time for the unstake request.

#### 2. Unstaking
   Unstaking is the process of withdrawing locked **TON** after meeting specific conditions, such as a predefined unlock time. The **Unstake Request** contract ensures secure fund management and timely release.

   ##### Unstaking Conditions
   Before the unstaking process can proceed, the following conditions must be satisfied:
   1. **Unlock Time Has Passed:**
        - The current blockchain time (`now()`) must be greater than or equal to the `unlock_timestamp`.
        - If not, the contract throws `ERROR::UNLOCK_TIMESTAMP_HAS_NOT_EXPIRED_YET`.

   2. **Sufficient Contract Balance**:
        - The contract must hold enough TON to cover:
            - The `ton_amount` requested for unstaking.
            - Any required fees (e.g., gas and message fees).
        - If the balance is insufficient, the contract may delay the unstaking process.

   3. **Valid Unstake Request**:
        - The request must have been deployed, and `unlock_timestamp` must not be `0`.
        - If not, the contract throws `ERROR::NOT_ALLOWED`.

   ##### Triggering the Unstake Operation
   - The user sends an **external** or an **internal** message to the contract, invoking the unstake operation.
   - The contract performs the following actions:
        1. Validates that the sender is the `owner_address`.
        2. Ensures all conditions for unstaking are met.
        3. Calculates the unstakable TON and bmTON amounts, considering fees.

   ##### Transfer of Funds
   Once the unstaking conditions are met:
   1. The contract creates a payload for the **Financial** contract using `FIN_OP::UNSTAKE` and sends message to it.
   2. **Financial** contract sends back TON to user (`unstake_amount - storage_fee`) and message with `OP::UNSTAKE_NOTIFICATION`.
   ###### TL-B scheme:
   ```
   unstake#492ab1b3 index:uint64 owner_address:MsgAddressInt ton_amount:Coins jetton_amount:Coins forward_payload:(Maybe ^Cell) = InternalMsgBody;
   ```
   - `index`: Unique identifier for the unstake request.
   - `owner_address`: Address of the unstake requester.
   - `ton_amount`: Amount of TON to be unstaked.
   - `jetton_amount`: Amount of jettons to be unstaked (currently unused).
   - `forward_payload`: Optional additional data for the unstake operation.

   ##### Reset Unstake Request
   After the funds are transferred the `unlock_timestamp` is reset to `0`, marking the unstake request as completed.
   This ensures the unstaking process cannot be repeated for the same request.

#### Get-methods
- `get_unstake_data`
    Returns contract data.
       

## Jetton Wallet

### Overview
This contract implements a bmTON jetton wallet, which allows transferring, receiving, and burning jettons. The wallet operates under the TON Blockchain standard for fungible tokens.

### Storage Variables
- `balance`: The current bmTON balance.
- `owner_address`: The wallet owner's address.
- `jetton_master_address`: The bmTON master contract's address.

#### TL-B scheme of storage
```
_ balance:Coins owner_address:MsgAddressInt jetton_master_address:MsgAddressInt = JettonWalletStorage;
```

### Errors
- `ERROR::NOT_FROM_JETTON_MASTER` (704): Triggered when a message is received from an address that is not the bmTON master address.
- `ERROR::NOT_FROM_OWNER` (705): Triggered when a sender other than the wallet owner attempts an action restricted to the owner.
- `ERROR::INSUFFICIENT_JETTON_BALANCE` (706): Triggered when the wallet’s bmTON balance is insufficient to complete the requested operation.
- `ERROR::NOT_FROM_JETTON_MASTER_OR_WALLET` (707): Triggered when a message is received from an address that is neither the bmTON master nor a valid bmTON wallet.
- `ERROR::EMPTY_FORWARD_PAYLOAD` (708): Triggered when a forward payload is expected but missing in the message.
- `ERROR::WRONG_OP` (709): Triggered when an unsupported or invalid operation code (op) is encountered in the message.
- `ERROR::INVALID_RECEIVER_ADDRESS` (710): Triggered when the receiver address for a transfer or burn operation is invalid or uninitialized.
- `ERROR::UNKNOWN_OP` (0xffff):     Triggered when an operation code (op) is unrecognized in the computation phase.

### Functionality
- `OP::TRANSFER`
    Initiates an outgoing transfer to a specified recipient.
    ##### TL-B scheme:
    ```
    transfer#0f8a7ea5 query_id:uint64 jetton_amount:Coins destination:MsgAddressInt
                    response_address:MsgAddressInt custom_payload:(Maybe ^Cell)
                    forward_ton_amount:Coins forward_payload:(Either Cell ^Cell)
                    = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
    - `jetton_amount`: Amount of jettons to be transferred.
    - `destination`: Address of the recipient.
    - `response_address`: Address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins.
    - `custom_payload`: Optional custom data (which is used by either sender or receiver jetton wallet for inner logic).
    - `forward_ton_amount`: The amount of nanotons to be sent to the destination address.
    - `forward_payload`: Optional custom data that should be sent to the destination address.

    **Should be rejected if:**
    1. Message is not from the owner.
    2. There is no enough jettons on the sender wallet
    3. There is no enough TON (with respect to jetton own storage fee guidelines and operation costs) to process operation, deploy receiver's jetton-wallet and send `forward_ton_amount`.
    4. After processing the request, the receiver's jetton-wallet **must** send at least `in_msg_value - forward_ton_amount - 2 * max_tx_gas_price - 2 * fwd_fee` to the `response_address` address.
    If the sender jetton-wallet cannot guarantee this, it must immediately stop executing the request and throw error.
    `max_tx_gas_price` is the price in Toncoins of maximum transaction gas limit of FT habitat workchain. For the basechain it can be obtained from [`ConfigParam 21`](https://github.com/ton-blockchain/ton/blob/78e72d3ef8f31706f30debaf97b0d9a2dfa35475/crypto/block/block.tlb#L660) from `gas_limit` field.  `fwd_fee` is forward fee for transfer request, it can be obtained from parsing transfer request message.
    5. There's no forward payload int the message.

    **Otherwise should do:**
    1. Decrease jetton amount on sender wallet by `jetton_amount` and send message which increase jetton amount on receiver wallet (and optionally deploy it).
    2. If `forward_amount > 0` ensure that receiver's jetton-wallet send message to `destination` address with `forward_amount` nanotons attached and with the following layout:

- `OP::INTERNAL_TRANSFER`
    Processes an incoming transfer from another wallet or bmTON jetton master.
    ##### TL-B scheme:
    ```
    internal_transfer#178d4519 query_id:uint64 jetton_amount:Coins from:MsgAddressInt
                    response_address:MsgAddressInt forward_ton_amount:Coins
                    forward_payload:(Either Cell ^Cell) = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
    - `jetton_amount`: Amount of transferred jettons.
    - `from`: Address of the sender (transfer initiator).
    - `response_address`: Address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins.
    - `forward_ton_amount`: The amount of nanotons to be sent to the destination address.
    - `forward_payload`: Optional custom data that should be sent to the destination address.
    
    **Should be rejected if:**
    - Message is not from a valid jetton wallet or the jetton master.

    **Otherwise should do:**
    1. Increase jetton amount on sender wallet by `jetton_amount` and send message which increase jetton amount on receiver wallet (and optionally deploy it).
    2. If `forward_ton_amount` is not zero, send message with the following layout:
    TL-B scheme: 
    ```
    transfer_notification#7362d09c query_id:uint64 amount:Coins sender:MsgAddressInt forward_payload:(Either Cell ^Cell) = InternalMsgBody;
    ```
    to the wallet owner. `query_id` should be equal with request's `query_id`.
    3. If `response_address` is not none, send message with the following layout:
    TL-B scheme: 
    ```
    excesses#d53276db query_id:uint64 = InternalMsgBody;
    ```
    to the response address. `query_id` should be equal with request's `query_id`.

- `OP::BURN`
    Burns a specified amount of bmTON.
    ##### TL-B scheme:
    ```
    burn#595f07bc query_id:uint64 jetton_amount:Coins response_address:MsgAddressInt custom_payload:(Maybe ^Cell) = InternalMsgBody;
    ```
    - `query_id`: A 64-bit unsigned integer serving as the unique identifier for the request.
    - `jetton_amount`: Amount of burned jettons.
    - `response_address`: Address where to send a response with confirmation of a successful burn and the rest of the incoming message coins.
    - `custom_payload`: Optional custom data.
    
    **Should be rejected if:**
    1. Message is not from the owner.
    2. There is no enough jettons on the sender wallet
    3. There is no enough TONs to send after processing the request at least `in_msg_value -  max_tx_gas_price` to the `response_address` address.
    If the sender jetton-wallet cannot guarantee this, it must immediately stop executing the request and throw error.

    **Otherwise should do:**
    1. Decrease jetton amount on burner wallet by `jetton_amount` and send notification to jetton master with information about burn.
    2. Jetton master should send all excesses of incoming message coins to `response_address` with the following layout:
    TL-B scheme:
    ```
    excesses#d53276db query_id:uint64 = InternalMsgBody;
    ```
    `query_id` should be equal with request's `query_id`.

- `OP::RETURN_TON`
    Returns all TON from the wallet to the owner, excluding only the minimal amount required for storage.
    ##### TL-B scheme:
    ```
    return_ton#054fa365 = InternalMsgBody;
    ```

#### Get-methods
- `get_wallet_data`:
    Retrieves key information about the bmTON jetton wallet.
    ##### Returns
    - `balance`: The current bmTON balance of the wallet.
    - `owner_address`: The owner address of the wallet.
    - `jetton_master_address`: The address of the bmTON master contract that controls this wallet.
    Only this address can issue or manage bmTON for this wallet.
    - `my_code()`: The cell containing the smart contract code.
    ##### TL-B scheme:
    ```
    get_wallet_data#_ = (balance:Coins owner_address:MsgAddressInt jetton_master_address:MsgAddressInt jetton_wallet_code:^Cell)
    ```