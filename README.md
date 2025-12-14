# CIS375
ClassProject
import sqlite3
from typing import List, Dict, Any, Optional

def get_db():
    return sqlite3.connect("vegetables.db", check_same_thread=False)

def add_vegetable(name, quantity, date_harvested, status, farmer_id):
    conn = get_db()
    cur = conn.cursor()
    cur.execute("INSERT INTO vegetables (name, quantity, date_harvested, status, farmer_id) VALUES (?, ?, ?, ?, ?)",
                (name, quantity, date_harvested, status, farmer_id))
    conn.commit()
    conn.close()

def update_vegetable(veg_id, name, quantity, date_harvested, status):
    conn = get_db()
    cur = conn.cursor()
    cur.execute("UPDATE vegetables SET name=?, quantity=?, date_harvested=?, status=? WHERE id=?",
                (name, quantity, date_harvested, status, veg_id))
    conn.commit()
    conn.close()

def delete_vegetable(veg_id):
    conn = get_db()
    cur = conn.cursor()
    cur.execute("DELETE FROM vegetables WHERE id=?", (veg_id,))
    conn.commit()
    conn.close()

def get_vegetables(role, user_id=None):
    conn = get_db()
    cur = conn.cursor()
    if role == 'farmer':
        cur.execute("SELECT * FROM vegetables WHERE farmer_id=?", (user_id,))
    else:
        cur.execute("SELECT * FROM vegetables WHERE quantity > 0")
    rows = cur.fetchall()
    conn.close()
    return rows

def log_activity(user_id, action, details):
    conn = get_db()
    cur = conn.cursor()
    cur.execute("INSERT INTO activity_log (user_id, action, details) VALUES (?, ?, ?)", (user_id, action, details))
    conn.commit()
    conn.close()

import gradio as gr
from datetime import datetime

# User authentication mockup
def authenticate(username, password):
    # Implement real authentication! Here, just an example:
    if username == "farmer" and password == "farm123":
        return {"role": "farmer", "user_id": 1}
    if username == "customer" and password == "cust123":
        return {"role": "customer", "user_id": 2}
    return None

def dashboard(role, user_id):
    # Get veg list for farmer or customer
    vegs = get_vegetables(role, user_id)
    columns = ["ID", "Name", "Quantity", "Date Harvested", "Status", "Farmer ID"]
    return vegs, columns

def add_veg(name, qty, date_harvested, user):
    status = "fresh" if int(qty) > 10 else "low stock"
    add_vegetable(name, int(qty), date_harvested, status, user["user_id"])
    log_activity(user["user_id"], "add", f"Added {name}, {qty}")
    return gr.update(value="Successfully added!", visible=True), dashboard(user["role"], user["user_id"])

def edit_veg(row, user):
    veg_id, name, qty, date_harvested, status, _ = row
    update_vegetable(veg_id, name, int(qty), date_harvested, status)
    log_activity(user["user_id"], "edit", f"Updated {name} to {qty}")
    return gr.update(value="Update successful!", visible=True), dashboard(user["role"], user["user_id"])

def delete_veg(row, user):
    veg_id = row[0]
    delete_vegetable(veg_id)
    log_activity(user["user_id"], "delete", f"Deleted vegetable ID {veg_id}")
    return gr.update(value="Deleted!", visible=True), dashboard(user["role"], user["user_id"])

# Gradio components & interface
with gr.Blocks(theme=gr.themes.Default(), css=".gradio-container { font-size: 18px; }") as demo:
    gr.Markdown("# 🥦 Fresh Vegetable Tracker")
    user_state = gr.State()
    
    with gr.Row():
        username = gr.Textbox(label="Username")
        password = gr.Textbox(label="Password", type="password")
        login_btn = gr.Button("Login")
        login_message = gr.Textbox(visible=False, interactive=False)
    
    def login_callback(username, password):
        user = authenticate(username, password)
        if user:
            return gr.update(visible=True), user, ""
        else:
            return gr.update(visible=False), None, "Login failed!"

    login_btn.click(login_callback, [username, password], [login_message, user_state, login_message])
    
    with gr.Tab("Dashboard") as dashboard_tab:
        veg_table = gr.Dataframe(headers=["ID", "Name", "Quantity", "Date Harvested", "Status", "Farmer ID"], interactive=True)
        success_box = gr.Textbox(visible=False, interactive=False)
        
        add_name = gr.Textbox(label="Vegetable Name")
        add_qty = gr.Number(label="Quantity")
        add_date = gr.Textbox(label="Date Harvested (YYYY-MM-DD)", value=datetime.today().date())
        add_btn = gr.Button("Add/Restock Vegetable")
        
        # Functions to bind to Gradio events
        add_btn.click(add_veg, [add_name, add_qty, add_date, user_state], [success_box, veg_table])
        veg_table.edit(edit_veg, [veg_table, user_state], [success_box, veg_table])
        veg_table.delete(delete_veg, [veg_table, user_state], [success_box, veg_table])
        
        # ...Repeat similar blocks for viewing activity logs, notifications, etc...

    # Feedback
    with gr.Tab("Feedback"):
        feedback_box = gr.Textbox(label="Submit your feedback here:", lines=3)
        feedback_btn = gr.Button("Send Feedback")
        feedback_thankyou = gr.Textbox(visible=False, interactive=False)
        def submit_feedback(text, user):
            if user:
                log_activity(user["user_id"], "feedback", text)
                return gr.update(value="Thanks for your feedback!", visible=True)
            else:
                return gr.update(value="Login to submit feedback.", visible=True)
        feedback_btn.click(submit_feedback, [feedback_box, user_state], feedback_thankyou)
