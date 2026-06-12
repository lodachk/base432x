# base432x
0xC1fC064c2EF5F8186c8467296764008C2E84F12B
import time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

DEPLOY_WINDOW = 15
ALLOCATION_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting early allocation wallets...\n")

    last_block = w3.eth.block_number

    deployed_tokens = {}

    wallet_allocations = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block > last_block:

                # detect contract creations
                for block_num in range(
                    last_block + 1,
                    current_block + 1
                ):

                    block = w3.eth.get_block(
                        block_num,
                        full_transactions=True
                    )

                    for tx in block.transactions:

                        if tx.to is None:

                            receipt = (
                                w3.eth.get_transaction_receipt(
                                    tx.hash
                                )
                            )

                            if receipt.contractAddress:
                                deployed_tokens[
                                    receipt.contractAddress
                                ] = block_num


                logs = w3.eth.get_logs({
                    "fromBlock": last_block + 1,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                for log in logs:

                    token = log["address"]

                    if token not in deployed_tokens:
                        continue

                    age = (
                        log["blockNumber"]
                        - deployed_tokens[token]
                    )

                    # very early token distribution
                    if age <= DEPLOY_WINDOW:

                        receiver = decode_address(
                            log["topics"][2]
                        )

                        sender = decode_address(
                            log["topics"][1]
                        )

                        if (
                            receiver != ZERO
                            and sender != ZERO
                        ):
                            wallet_allocations[receiver] += 1


                print(
                    f"\nBlocks "
                    f"{last_block}"
                    f" -> {current_block}"
                )


                for wallet, count in wallet_allocations.items():

                    if count >= ALLOCATION_THRESHOLD:

                        print("🎁 Early Allocation Wallet")
                        print("Wallet:", wallet)
                        print("Allocations:", count)
                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
