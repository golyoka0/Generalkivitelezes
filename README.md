import streamlit as st
import sqlite3
from datetime import date, datetime
from io import BytesIO
import html

st.set_page_config(page_title="Generálkivitelezői árajánlatkészítő", page_icon="🏗️", layout="wide")

DB = "arajanlatok.db"

def db():
    con = sqlite3.connect(DB, check_same_thread=False)
    con.row_factory = sqlite3.Row
    return con

def init_db():
    with db() as c:
        c.executescript("""
        CREATE TABLE IF NOT EXISTS materials(
            id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, unit TEXT NOT NULL,
            cost REAL NOT NULL DEFAULT 0, price REAL NOT NULL DEFAULT 0, note TEXT DEFAULT ''
        );
        CREATE TABLE IF NOT EXISTS works(
            id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, unit TEXT NOT NULL,
            cost REAL NOT NULL DEFAULT 0, price REAL NOT NULL DEFAULT 0, note TEXT DEFAULT ''
        );
        CREATE TABLE IF NOT EXISTS customers(
            id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, email TEXT DEFAULT '',
            phone TEXT DEFAULT '', billing TEXT DEFAULT '', site TEXT DEFAULT ''
        );
        CREATE TABLE IF NOT EXISTS quotes(
            id INTEGER PRIMARY KEY AUTOINCREMENT, number TEXT UNIQUE, customer_id INTEGER,
            project TEXT DEFAULT '', created TEXT, valid_until TEXT, vat REAL DEFAULT 27,
            status TEXT DEFAULT 'Piszkozat', note TEXT DEFAULT '', terms TEXT DEFAULT '',
            FOREIGN KEY(customer_id) REFERENCES customers(id)
        );
        CREATE TABLE IF NOT EXISTS quote_lines(
            id INTEGER PRIMARY KEY AUTOINCREMENT, quote_id INTEGER, kind TEXT, description TEXT,
            unit TEXT, qty REAL, unit_cost REAL, unit_price REAL,
            FOREIGN KEY(quote_id) REFERENCES quotes(id)
        );
        CREATE TABLE IF NOT EXISTS settings(key TEXT PRIMARY KEY, value TEXT);
        """)
        defaults = {
            "company_name": "Saját vállalkozás",
            "company_address": "",
            "company_tax": "",
            "company_email": "",
            "company_phone": "",
            "default_vat": "27",
            "default_markup": "20",
            "quote_prefix": "AJ"
        }
        for k, v in defaults.items():
            c.execute("INSERT OR IGNORE INTO settings(key,value) VALUES(?,?)", (k,v))

def setting(key, default=""):
    with db() as c:
        r = c.execute("SELECT value FROM settings WHERE key=?", (key,)).fetchone()
    return r["value"] if r else default

def money(x):
    return f"{x:,.0f} Ft".replace(",", " ")

def rows(query, args=()):
    with db() as c:
        return c.execute(query, args).fetchall()

def save_setting(key, value):
    with db() as c:
        c.execute("INSERT INTO settings(key,value) VALUES(?,?) ON CONFLICT(key) DO UPDATE SET value=excluded.value",(key,str(value)))

init_db()

st.title("🏗️ Generálkivitelezői árajánlatkészítő")
st.caption("Anyagok • Munkadíjak • Ügyfelek • Árajánlatok • Automatikus kalkuláció")

with st.sidebar:
    st.header("Navigáció")
    page = st.radio("Menüpont", ["Áttekintés", "Anyagkatalógus", "Munkadíj-katalógus", "Ügyfelek", "Új árajánlat", "Árajánlatok", "Beállítások"])
    st.divider()
    st.caption("Az adatok ebben az alkalmazásban helyi SQLite-adatbázisba kerülnek. Éles, többfelhasználós használathoz felhő-adatbázis és belépés beállítása szükséges.")

if page == "Áttekintés":
    q = rows("SELECT * FROM quotes ORDER BY id DESC")
    m = rows("SELECT * FROM materials")
    w = rows("SELECT * FROM works")
    cu = rows("SELECT * FROM customers")
    a,b,c,d = st.columns(4)
    a.metric("Árajánlatok", len(q))
    b.metric("Anyagok", len(m))
    c.metric("Munkafolyamatok", len(w))
    d.metric("Ügyfelek", len(cu))
    st.subheader("Legutóbbi árajánlatok")
    if q:
        st.dataframe([dict(r) for r in q[:8]], use_container_width=True, hide_index=True)
    else:
        st.info("Még nincs árajánlat. Kezdd az anyagok, munkadíjak vagy ügyfelek felvitelével.")

elif page in ("Anyagkatalógus", "Munkadíj-katalógus"):
    is_mat = page == "Anyagkatalógus"
    table = "materials" if is_mat else "works"
    st.subheader("Új tétel felvétele")
    with st.form("catalog_form", clear_on_submit=True):
        name = st.text_input("Megnevezés")
        col1,col2,col3 = st.columns(3)
        unit = col1.selectbox("Mértékegység", ["db","m","m²","m³","óra","nap","kg","t","csomag","fm"])
        cost = col2.number_input("Nettó beszerzési / önköltség (Ft)", min_value=0.0, step=100.0)
        price = col3.number_input("Nettó ügyfélár (Ft)", min_value=0.0, step=100.0)
        note = st.text_input("Megjegyzés")
        submitted = st.form_submit_button("Tétel mentése", type="primary")
        if submitted:
            if not name.strip():
                st.error("A megnevezés kötelező.")
            else:
                with db() as cx:
                    cx.execute(f"INSERT INTO {table}(name,unit,cost,price,note) VALUES(?,?,?,?,?)",(name.strip(),unit,cost,price,note))
                st.success("Tétel elmentve.")
                st.rerun()
    st.subheader("Katalógus")
    search = st.text_input("Keresés")
    data = rows(f"SELECT * FROM {table} WHERE name LIKE ? ORDER BY name", (f"%{search}%",))
    if data:
        st.dataframe([dict(x) for x in data], use_container_width=True, hide_index=True)
        opts = {f'{x["id"]} – {x["name"]}': x["id"] for x in data}
        selected = st.selectbox("Törlendő tétel", ["—"] + list(opts))
        if selected != "—" and st.button("Kiválasztott tétel törlése"):
            with db() as cx: cx.execute(f"DELETE FROM {table} WHERE id=?", (opts[selected],))
            st.rerun()
    else:
        st.info("Nincs megjeleníthető tétel.")

elif page == "Ügyfelek":
    st.subheader("Új ügyfél")
    with st.form("customer_form", clear_on_submit=True):
        name = st.text_input("Ügyfél / cég neve *")
        email = st.text_input("E-mail")
        phone = st.text_input("Telefon")
        billing = st.text_area("Számlázási cím")
        site = st.text_area("Munkavégzés helye")
        if st.form_submit_button("Ügyfél mentése", type="primary"):
            if not name.strip(): st.error("A név kötelező.")
            else:
                with db() as cx:
                    cx.execute("INSERT INTO customers(name,email,phone,billing,site) VALUES(?,?,?,?,?)",(name,email,phone,billing,site))
                st.success("Ügyfél elmentve."); st.rerun()
    st.subheader("Ügyféllista")
    data = rows("SELECT * FROM customers ORDER BY name")
    if data:
        st.dataframe([dict(x) for x in data], use_container_width=True, hide_index=True)
        opts = {f'{x["id"]} – {x["name"]}': x["id"] for x in data}
        chosen = st.selectbox("Törlendő ügyfél", ["—"] + list(opts))
        if chosen != "—" and st.button("Ügyfél törlése"):
            with db() as cx: cx.execute("DELETE FROM customers WHERE id=?", (opts[chosen],))
            st.rerun()
    else: st.info("Még nincs ügyfél.")

elif page == "Új árajánlat":
    customers = rows("SELECT * FROM customers ORDER BY name")
    mats = rows("SELECT * FROM materials ORDER BY name")
    works = rows("SELECT * FROM works ORDER BY name")
    if not customers:
        st.warning("Először vigyél fel legalább egy ügyfelet az Ügyfelek menüpontban.")
    else:
        st.subheader("Árajánlat adatai")
        cust_opts = {f'{x["name"]} (#{x["id"]})': x for x in customers}
        cust_key = st.selectbox("Ügyfél", list(cust_opts))
        cust = cust_opts[cust_key]
        project = st.text_input("Projekt / munka megnevezése")
        col1,col2,col3 = st.columns(3)
        created = col1.date_input("Kiállítás dátuma", date.today())
        valid = col2.date_input("Érvényesség", date.today().replace(day=min(date.today().day,28)))
        vat = col3.number_input("ÁFA (%)", min_value=0.0, max_value=100.0, value=float(setting("default_vat","27")))
        st.subheader("Tételek")
        catalog_options = {}
        for x in mats: catalog_options[f"Anyag | {x['name']} ({x['unit']})"] = ("Anyag",x)
        for x in works: catalog_options[f"Munkadíj | {x['name']} ({x['unit']})"] = ("Munkadíj",x)
        chosen_items = st.multiselect("Katalógustételek", list(catalog_options))
        line_data = []
        for i, label in enumerate(chosen_items):
            kind, item = catalog_options[label]
            with st.expander(label, expanded=True):
                c1,c2,c3 = st.columns(3)
                qty = c1.number_input("Mennyiség", min_value=0.0, value=1.0, step=1.0, key=f"qty_{item['id']}_{kind}")
                cost = c2.number_input("Önköltség / egység (Ft)", min_value=0.0, value=float(item["cost"]), step=100.0, key=f"cost_{item['id']}_{kind}")
                price = c3.number_input("Ügyfélár / egység (Ft)", min_value=0.0, value=float(item["price"]), step=100.0, key=f"price_{item['id']}_{kind}")
                line_data.append({"kind":kind,"description":item["name"],"unit":item["unit"],"qty":qty,"cost":cost,"price":price})
        with st.expander("Egyedi tétel hozzáadása"):
            custom_name = st.text_input("Egyedi tétel megnevezése")
            c1,c2,c3,c4 = st.columns(4)
            custom_unit = c1.selectbox("Egység",["db","m","m²","m³","óra","nap","kg","t","fm"],key="custom_unit")
            custom_qty = c2.number_input("Mennyiség",min_value=0.0,value=1.0,key="custom_qty")
            custom_cost = c3.number_input("Önköltség / egység",min_value=0.0,key="custom_cost")
            custom_price = c4.number_input("Ügyfélár / egység",min_value=0.0,key="custom_price")
            if custom_name.strip():
                line_data.append({"kind":"Egyedi","description":custom_name.strip(),"unit":custom_unit,"qty":custom_qty,"cost":custom_cost,"price":custom_price})
        net = sum(x["qty"]*x["price"] for x in line_data)
        cost_total = sum(x["qty"]*x["cost"] for x in line_data)
        tax = net*vat/100
        gross = net+tax
        profit = net-cost_total
        a,b,c,d = st.columns(4)
        a.metric("Nettó összesen",money(net)); b.metric("ÁFA",money(tax)); c.metric("Bruttó",money(gross)); d.metric("Várható fedezet",money(profit))
        note = st.text_area("Megjegyzés")
        terms = st.text_area("Fizetési / teljesítési feltételek")
        if st.button("Árajánlat mentése",type="primary",disabled=not line_data):
            prefix=setting("quote_prefix","AJ")
            with db() as cx:
                cur=cx.execute("INSERT INTO quotes(number,customer_id,project,created,valid_until,vat,status,note,terms) VALUES(?,?,?,?,?,?,?,?,?)",
                    ("TEMP",cust["id"],project,created.isoformat(),valid.isoformat(),vat,"Piszkozat",note,terms))
                qid=cur.lastrowid
                number=f"{prefix}-{created.year}-{qid:04d}"
                cx.execute("UPDATE quotes SET number=? WHERE id=?",(number,qid))
                for x in line_data:
                    cx.execute("INSERT INTO quote_lines(quote_id,kind,description,unit,qty,unit_cost,unit_price) VALUES(?,?,?,?,?,?,?)",
                        (qid,x["kind"],x["description"],x["unit"],x["qty"],x["cost"],x["price"]))
            st.success(f"Elmentve: {number}")
            st.rerun()

elif page == "Árajánlatok":
    data=rows("""SELECT q.*, c.name AS customer_name FROM quotes q LEFT JOIN customers c ON c.id=q.customer_id ORDER BY q.id DESC""")
    search=st.text_input("Keresés ajánlatszám vagy ügyfél szerint")
    data=[x for x in data if search.lower() in (x["number"] or "").lower() or search.lower() in (x["customer_name"] or "").lower()]
    if not data: st.info("Nincs találat.")
    else:
        for q in data:
            lines=rows("SELECT * FROM quote_lines WHERE quote_id=?",(q["id"],))
            net=sum(x["qty"]*x["unit_price"] for x in lines)
            tax=net*q["vat"]/100
            with st.expander(f'{q["number"]} · {q["customer_name"]} · {money(net+tax)}'):
                st.write(f"Projekt: {q['project']} | Dátum: {q['created']} | ÁFA: {q['vat']}% | Állapot: {q['status']}")
                if lines:
                    st.dataframe([{"Típus":x["kind"],"Megnevezés":x["description"],"Mennyiség":x["qty"],"Egység":x["unit"],"Egységár":money(x["unit_price"]),"Nettó":money(x["qty"]*x["unit_price"])} for x in lines],use_container_width=True,hide_index=True)
                st.write(f"**Nettó:** {money(net)} · **ÁFA:** {money(tax)} · **Bruttó:** {money(net+tax)}")
                st.write(q["note"] or "")
                st.write(q["terms"] or "")
                col1,col2=st.columns(2)
                status=col1.selectbox("Státusz",["Piszkozat","Elküldve","Elfogadva","Elutasítva","Lezárva"],index=["Piszkozat","Elküldve","Elfogadva","Elutasítva","Lezárva"].index(q["status"]),key=f"status_{q['id']}")
                if col1.button("Státusz mentése",key=f"save_status_{q['id']}"):
                    with db() as cx: cx.execute("UPDATE quotes SET status=? WHERE id=?",(status,q["id"]))
                    st.rerun()
                if col2.button("Árajánlat törlése",key=f"del_quote_{q['id']}"):
                    with db() as cx:
                        cx.execute("DELETE FROM quote_lines WHERE quote_id=?",(q["id"],))
                        cx.execute("DELETE FROM quotes WHERE id=?",(q["id"],))
                    st.rerun()

elif page == "Beállítások":
    st.subheader("Vállalkozás adatai")
    fields=[("company_name","Vállalkozás neve"),("company_address","Székhely / cím"),("company_tax","Adószám"),("company_email","E-mail"),("company_phone","Telefonszám"),("quote_prefix","Ajánlatszám előtag")]
    with st.form("settings"):
        vals={}
        for key,label in fields: vals[key]=st.text_input(label,value=setting(key))
        vat=st.number_input("Alapértelmezett ÁFA (%)",min_value=0.0,max_value=100.0,value=float(setting("default_vat","27")))
        markup=st.number_input("Alapértelmezett árrés (%) – tájékoztató",min_value=0.0,value=float(setting("default_markup","20")))
        if st.form_submit_button("Beállítások mentése",type="primary"):
            for k,v in vals.items(): save_setting(k,v)
            save_setting("default_vat",vat); save_setting("default_markup",markup)
            st.success("Beállítások mentve.")

st.divider()
st.caption("Prototípus – használat előtt állítsd be a vállalkozás adatait, és teszteld a számításokat. Éles használathoz biztonságos belépés, felhő-adatbázis, biztonsági mentés és PDF-generálás szükséges.")
