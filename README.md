# base123ojj
0x9e4E1f2B064a489E48EB9b427fD0f742bdEb9e4C
import time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20

EARLY_ENTRY_LIMIT = 10
EXIT_WINDOW = 50

SCORE_THRESHOLD = 5

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting early exit wallets...\n")

    last_block = w3.eth.block_number

    wallet_entries = defaultdict(dict)

    token_activity = defaultdict(int)

    wallet_scores = defaultdict(int)


    while True:

        try:

            current_block = w3.eth.block_number


            if current_block >= last_block + WINDOW_BLOCKS:


                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                current_activity = defaultdict(int)


                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )


                    current_activity[token] += 1


                    if receiver != ZERO:

                        # early entry tracking
                        if token not in wallet_entries[receiver]:

                            if (
                                token_activity[token]
                                <= EARLY_ENTRY_LIMIT
                            ):

                                wallet_entries[receiver][token] = (
                                    log["blockNumber"]
                                )


                    if sender != ZERO:

                        if token in wallet_entries[sender]:

                            entered = (
                                wallet_entries[sender][token]
                            )

                            age = (
                                log["blockNumber"]
                                - entered
                            )


                            # exited while still early
                            if (
                                age <= EXIT_WINDOW
                                and current_activity[token]
                                <= EARLY_ENTRY_LIMIT
                            ):

                                wallet_scores[sender] += 1



                for token, activity in current_activity.items():

                    token_activity[token] = activity


                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, score in wallet_scores.items():

                    if score >= SCORE_THRESHOLD:

                        print("🏃 Early Exit Wallet")
                        print("Wallet:", wallet)
                        print("Score:", score)
                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)



if __name__ == "__main__":
    main()
