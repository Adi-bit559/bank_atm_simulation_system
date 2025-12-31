# Banking & ATM Simulation System

import json
import os
from datetime import datetime, date
from getpass import getpass

# ---------------- CONFIG -----------------
DATA_FILE = "bank_data.json"
MAX_PIN_ATTEMPTS = 3
DAILY_LIMIT = 50000
INTEREST_RATE = 0.04  # 4% annual

# --------------- COLOR UTILS -------------
class Color:
    GREEN = "\033[92m"
    RED = "\033[91m"
    YELLOW = "\033[93m"
    CYAN = "\033[96m"
    END = "\033[0m"

# --------------- DATA HANDLING -----------
def load_data():
    if not os.path.exists(DATA_FILE):
        return {"accounts": {}, "admin_pin": "0000"}
    with open(DATA_FILE, "r") as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)

# --------------- UTILITIES ---------------
def log_tx(account, tx_type, amount):
    account.setdefault("transactions", []).append({
        "date": str(datetime.now()),
        "type": tx_type,
        "amount": amount
    })
    account["transactions"] = account["transactions"][-10:]

# --------------- AUTH --------------------
def authenticate(data):
    acc = input("Account Number: ")
    if acc not in data["accounts"]:
        print(Color.RED + "Account not found" + Color.END)
        return None

    for i in range(MAX_PIN_ATTEMPTS):
        pin = getpass("Enter PIN: ")
        if pin == data["accounts"][acc]["pin"]:
            return acc
        print(Color.RED + "Wrong PIN" + Color.END)
    print(Color.RED + "Account locked" + Color.END)
    return None

# --------------- BANK OPS ----------------
def deposit(acc_data):
    amt = float(input("Amount: "))
    acc_data["balance"] += amt
    log_tx(acc_data, "Deposit", amt)

def withdraw(acc_data):
    amt = float(input("Amount: "))
    today = str(date.today())
    acc_data.setdefault("daily", {}).setdefault(today, 0)

    if acc_data["daily"][today] + amt > DAILY_LIMIT:
        print(Color.RED + "Daily limit exceeded" + Color.END)
        return
    if amt > acc_data["balance"]:
        print(Color.RED + "Insufficient balance" + Color.END)
        return

    acc_data["balance"] -= amt
    acc_data["daily"][today] += amt
    log_tx(acc_data, "Withdraw", amt)

def transfer(data, from_acc):
    to = input("To Account: ")
    amt = float(input("Amount: "))
    if to not in data["accounts"]:
        print(Color.RED + "Invalid account" + Color.END)
        return
    if amt > data["accounts"][from_acc]["balance"]:
        print(Color.RED + "Insufficient balance" + Color.END)
        return

    data["accounts"][from_acc]["balance"] -= amt
    data["accounts"][to]["balance"] += amt
    log_tx(data["accounts"][from_acc], "Transfer Out", amt)
    log_tx(data["accounts"][to], "Transfer In", amt)

# --------------- STATEMENTS --------------
def mini_statement(acc_data):
    print(Color.CYAN + "--- Mini Statement ---" + Color.END)
    for tx in acc_data.get("transactions", []):
        print(tx)

# --------------- INTEREST ----------------
def apply_interest(acc_data):
    interest = acc_data["balance"] * INTEREST_RATE
    acc_data["balance"] += interest
    log_tx(acc_data, "Interest", interest)

# --------------- ADMIN -------------------
def admin_panel(data):
    pin = getpass("Admin PIN: ")
    if pin != data["admin_pin"]:
        print(Color.RED + "Access denied" + Color.END)
        return

    while True:
        print("1.Create  2.Delete  3.Exit")
        ch = input("Choice: ")
        if ch == "1":
            acc = input("Account No: ")
            data["accounts"][acc] = {
                "pin": input("PIN: "),
                "balance": 0,
                "transactions": []
            }
        elif ch == "2":
            acc = input("Account No: ")
            data["accounts"].pop(acc, None)
        else:
            break

# --------------- MAIN MENU ---------------
def atm_menu(data, acc):
    acc_data = data["accounts"][acc]
    while True:
        print(Color.YELLOW + "\n1.Deposit 2.Withdraw 3.Transfer 4.Balance 5.Statement 6.Interest 7.Exit" + Color.END)
        ch = input("Choice: ")
        if ch == "1": deposit(acc_data)
        elif ch == "2": withdraw(acc_data)
        elif ch == "3": transfer(data, acc)
        elif ch == "4": print("Balance:", acc_data["balance"])
        elif ch == "5": mini_statement(acc_data)
        elif ch == "6": apply_interest(acc_data)
        else: break

# --------------- ENTRY -------------------
def main():
    data = load_data()
    while True:
        print("1.User 2.Admin 3.Exit")
        ch = input("Choice: ")
        if ch == "1":
            acc = authenticate(data)
            if acc: atm_menu(data, acc)
        elif ch == "2": admin_panel(data)
        else: break
        save_data(data)

if __name__ == "__main__":
    main()
