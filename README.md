import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime
import os
from docx import Document  # Make sure python-docx is installed

# ---------------- DATABASE ----------------
conn = sqlite3.connect("bar_tab.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS customers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    customer_name TEXT NOT NULL,
    amount REAL NOT NULL,
    type TEXT NOT NULL,
    date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
""")
conn.commit()

# ---------------- FUNCTIONS ----------------

def add_customer(name):
    if name.strip() == "":
        messagebox.showerror("Error", "Enter a valid customer name")
        return
    try:
        cursor.execute("INSERT INTO customers (name) VALUES (?)", (name,))
        conn.commit()
        update_customer_list()
        entry_new_customer.delete(0, tk.END)
    except sqlite3.IntegrityError:
        messagebox.showinfo("Info", f"{name} is already a regular.")

def add_purchase():
    name = customer_var.get()
    if not name:
        messagebox.showerror("Error", "Select a customer")
        return
    try:
        amount = float(entry_amount.get())
    except ValueError:
        messagebox.showerror("Error", "Enter a valid number")
        return
    cursor.execute("""
    INSERT INTO transactions (customer_name, amount, type)
    VALUES (?, ?, 'purchase')
    """, (name, amount))
    conn.commit()
    entry_amount.delete(0, tk.END)
    show_balance()

def add_payment():
    name = customer_var.get()
    if not name:
        messagebox.showerror("Error", "Select a customer")
        return
    try:
        amount = float(entry_amount.get())
    except ValueError:
        messagebox.showerror("Error", "Enter a valid number")
        return
    cursor.execute("""
    INSERT INTO transactions (customer_name, amount, type)
    VALUES (?, ?, 'payment')
    """, (name, amount))
    conn.commit()
    entry_amount.delete(0, tk.END)
    show_balance()

def get_balance(name):
    cursor.execute("""
    SELECT 
        SUM(CASE WHEN type='purchase' THEN amount ELSE 0 END) -
        SUM(CASE WHEN type='payment' THEN amount ELSE 0 END)
    FROM transactions
    WHERE customer_name=?
    """, (name,))
    result = cursor.fetchone()[0]
    return result if result else 0

def show_balance():
    name = customer_var.get()
    if name:
        balance = get_balance(name)
        lbl_balance.config(text=f"Current Balance: R{balance:.2f}")
    else:
        lbl_balance.config(text="Current Balance: R0.00")

def create_invoice():
    name = customer_var.get()
    if not name:
        messagebox.showerror("Error", "Select a customer")
        return

    total = get_balance(name)
    if total == 0:
        messagebox.showinfo("Info", "Balance is already zero.")
        return

    # ---------------- Save invoice to C:\sTUFF ----------------
    folder_path = r"C:\sTUFF"
    if not os.path.exists(folder_path):
        os.makedirs(folder_path)

    # Load your existing template
    template_path = "Invoice_Template.docx"  # Place your template in same folder as EXE
    doc = Document(template_path)

    # Replace placeholders
    for p in doc.paragraphs:
        if "{CUSTOMER}" in p.text:
            p.text = p.text.replace("{CUSTOMER}", name)
        if "{TOTAL}" in p.text:
            p.text = p.text.replace("{TOTAL}", f"R{total:.2f}")
        if "{DATE}" in p.text:
            p.text = p.text.replace("{DATE}", datetime.now().strftime("%Y-%m-%d %H:%M"))

    # Save to C:\sTUFF
    filename = os.path.join(folder_path, f"Invoice_{name}_{datetime.now().strftime('%Y%m%d%H%M%S')}.docx")
    doc.save(filename)

    # ---------------- Reset tab for this customer ----------------
    cursor.execute("""
    INSERT INTO transactions (customer_name, amount, type)
    VALUES (?, ?, 'payment')
    """, (name, total))
    conn.commit()
    show_balance()

    messagebox.showinfo("Invoice Created",
                        f"Invoice saved in {folder_path}\n{name}'s tab has been reset to R0.00")

def update_customer_list():
    cursor.execute("SELECT name FROM customers ORDER BY name")
    customers = [row[0] for row in cursor.fetchall()]
    customer_combo['values'] = customers

# ---------------- GUI ----------------
root = tk.Tk()
root.title("MIKD Bar Tab System v1.0")
root.geometry("450x400")

tk.Label(root, text="Select Customer").pack(pady=5)
customer_var = tk.StringVar()
customer_combo = ttk.Combobox(root, textvariable=customer_var)
customer_combo.pack(pady=5)

entry_new_customer = tk.Entry(root)
entry_new_customer.pack(pady=5)
tk.Button(root, text="Add New Regular", 
          command=lambda: add_customer(entry_new_customer.get())).pack(pady=5)

tk.Label(root, text="Amount (R)").pack(pady=5)
entry_amount = tk.Entry(root)
entry_amount.pack(pady=5)

tk.Button(root, text="Add Purchase", command=add_purchase).pack(pady=5)
tk.Button(root, text="Add Payment", command=add_payment).pack(pady=5)
tk.Button(root, text="Generate Invoice", command=create_invoice).pack(pady=5)

lbl_balance = tk.Label(root, text="Current Balance: R0.00", font=("Helvetica", 14))
lbl_balance.pack(pady=10)



update_customer_list()
root.mainloop()
conn.close()

footer_frame = tk.Frame(root, bg="#111111")
footer_frame.pack(side="bottom", fill="x")

left_label = tk.Label(
    footer_frame,
    text="Bar Tab System v1.0",
    font=("Helvetica", 8),
    fg="white",
    bg="#111111"
)
left_label.pack(side="left", padx=10, pady=4)

right_label = tk.Label(
    footer_frame,
    text="Developed by MIKD © 2026",
    font=("Helvetica", 8),
    fg="white",
    bg="#111111"
)
right_label.pack(side="right", padx=10, pady=4)
