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
