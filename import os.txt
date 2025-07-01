import os
import time
import random
from web3 import Web3
from dotenv import load_dotenv

load_dotenv()
RPC = "https://rpc.plume.org"
CHAIN_ID = 98866
PRIVATE_KEY = os.getenv("PRIVATE_KEY")
web3 = Web3(Web3.HTTPProvider(RPC))
account = web3.eth.account.from_key(PRIVATE_KEY)
wallet = account.address

ROUTER = Web3.to_checksum_address("0xAaAaAAAA81a99d2a05eE428eC7a1d8A3C2237D85")
GAS_LIMIT = 200000
POOL_IDX = 420
LIMIT_PRICE = 2126743031358024713652501817180651137  # dummy value

TOKENS = {
    "USDC": {"addr": "0x78adD880A697070c1e765Ac44D65323a0DcCE913", "decimals": 6},
    "USDT": {"addr": "0xda6087E69C51E7D31b6DBAD276a3c44703DFdCAd", "decimals": 6},
    "pUSD": {"addr": "0xdddD73F5Df1F0DC31373357beAC77545dC5A6f3F", "decimals": 6},
}

ABI_SWAP = [{
    "name": "swap",
    "type": "function",
    "inputs": [
        {"name": "base", "type": "address"},
        {"name": "quote", "type": "address"},
        {"name": "poolIdx", "type": "uint256"},
        {"name": "isBuy", "type": "bool"},
        {"name": "inBaseQty", "type": "bool"},
        {"name": "qty", "type": "uint128"},
        {"name": "tip", "type": "uint16"},
        {"name": "limitPrice", "type": "uint128"},
        {"name": "minOut", "type": "uint128"},
        {"name": "reserveFlags", "type": "uint8"},
    ],
    "outputs": [],
    "stateMutability": "nonpayable"
}]

ABI_ERC20 = [
    {
        "name": "approve",
        "type": "function",
        "inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}],
        "outputs": [{"type": "bool"}],
        "stateMutability": "nonpayable"
    },
    {
        "name": "allowance",
        "type": "function",
        "inputs": [{"name": "owner", "type": "address"}, {"name": "spender", "type": "address"}],
        "outputs": [{"type": "uint256"}],
        "stateMutability": "view"
    },
]

def approve_if_needed(token_address, decimals):
    try:
        contract = web3.eth.contract(address=Web3.to_checksum_address(token_address), abi=ABI_ERC20)
        allowance = contract.functions.allowance(wallet, ROUTER).call()
        if allowance < 1e9:
            print(f"[APPROVE] Token {token_address} belum diapprove. Kirim approve...")
            nonce = web3.eth.get_transaction_count(wallet, "pending")
            tx = contract.functions.approve(ROUTER, 2**256 - 1).build_transaction({
                "from": wallet,
                "nonce": nonce,
                "gas": 60000,
                "gasPrice": web3.eth.gas_price,
                "chainId": CHAIN_ID
            })
            signed_tx = web3.eth.account.sign_transaction(tx, PRIVATE_KEY)
            tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)
            print(f"[APPROVE] TX: {web3.to_hex(tx_hash)}")
            web3.eth.wait_for_transaction_receipt(tx_hash)
        else:
            print(f"[APPROVE] Token {token_address} sudah diapprove.")
    except Exception as e:
        print(f"[ERROR] Gagal approve token {token_address}: {e}")

def get_random_amount(decimals):
    amount = random.uniform(0.01, 0.02)
    return int(amount * (10 ** decimals))

def get_min_out(decimals):
    return int(0.000000000001 * (10 ** decimals))

def swap_token(from_name, to_name):
    try:
        if from_name == to_name:
            return

        base = Web3.to_checksum_address(TOKENS[from_name]["addr"])
        quote = Web3.to_checksum_address(TOKENS[to_name]["addr"])
        decimals = TOKENS[from_name]["decimals"]
        qty = get_random_amount(decimals)
        min_out = get_min_out(TOKENS[to_name]["decimals"])
        approve_if_needed(base, decimals)
        nonce = web3.eth.get_transaction_count(wallet, "pending")

        contract = web3.eth.contract(address=ROUTER, abi=ABI_SWAP)
        tx = contract.functions.swap(
            base,
            quote,
            POOL_IDX,
            True,
            True,
            qty,
            0,
            LIMIT_PRICE,
            min_out,
            0
        ).build_transaction({
            "from": wallet,
            "nonce": nonce,
            "gas": GAS_LIMIT,
            "gasPrice": web3.eth.gas_price,
            "chainId": CHAIN_ID
        })

        signed_tx = web3.eth.account.sign_transaction(tx, PRIVATE_KEY)
        tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)
        print(f"[SWAP] {from_name} → {to_name} | Qty: {qty / (10 ** decimals):.8f} | TX: {web3.to_hex(tx_hash)}")
        web3.eth.wait_for_transaction_receipt(tx_hash)
    except Exception as e:
        print(f"[ERROR] Swap {from_name} → {to_name}: {e}")

def loop_swap_all():
    tokens = list(TOKENS.keys())
    while True:
        pairs = [(a, b) for a in tokens for b in tokens if a != b]
        random.shuffle(pairs)  # acak urutan

        for from_token, to_token in pairs:
            swap_token(from_token, to_token)
            delay = random.randint(40, 70)
            print(f"[DELAY] Menunggu {delay} detik sebelum swap berikutnya...\n")
            time.sleep(delay)

if __name__ == "__main__":
    print(f"🔁 Mulai swap antar token (acak + delay) | Wallet: {wallet}")
    loop_swap_all()
