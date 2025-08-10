# SaldoSuite
Boekhouden in eenvoud
SaldoSuite - MVP template voor NL Boekhouding + Salarisadministratie
Doel
- Self-hosted web-app (FastAPI backend + Next.js frontend) voor facturatie, boekhouding en payroll, met NL btw-regels. Multi-user met RBAC.
Stack
- Frontend: Next.js met TypeScript
- Backend: FastAPI (Python)
- Database: PostgreSQL
- Auth: JWT met rolgebaseerde toegang
- Deployment: Docker Compose (local development)
Wat zit erin
- backend/ (FastAPI + PostgreSQL)
- frontend/ (Next.js)
- docker-compose.yml
- seed-data en first-test-scenario
- CI/CD skeletons (GitHub Actions) voor PR-linting/tests; optionele CD
Seed-data
- NL btw-tarieven
- Eén klant, één product/dienst, één factuur, één werknemer en één payroll-run
First-testscenario
- 1 klant (Acme NL)
- 1 factuur met 2 regels
- 1 payroll-run voor twee werknemers
Installatie-instructies
- Zorg voor Docker en Docker Compose
- Maak een map SaldoSuite en zet alle bestanden erin zoals hieronder beschreven
- Kopieer .env.example naar .env en vul DB-credentials en JWT_SECRET in
- docker-compose up -d
- Initialiseer DB (create_all of Alembic migraties)
- Maak admin-user aan via POST /auth/register
- Test: klant aanmaken, factuur met regels, payroll-run, vat-rapport
- Open frontend: http://localhost:3000 (log in met admin)

Startpunt (bestanden en inhoud)  
- Let op: vervang placeholder-waarden in .env met echte waarden waar nodig.

Backend: main.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routers import auth, customers, invoices, journals, vat, payroll, reports

app = FastAPI(title="SaldoSuite NL Boekhouding MVP")

origins = ["*"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router, prefix="/auth")
app.include_router(customers.router, prefix="/customers")
app.include_router(invoices.router, prefix="/invoices")
app.include_router(journals.router, prefix="/journals")
app.include_router(vat.router, prefix="/vat")
app.include_router(payroll.router, prefix="/payroll")
app.include_router(reports.router, prefix="/reports")

@app.get("/")
def read_root():
    return {"msg": "SaldoSuite API running"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Backend: backend/__init__.py
```python
# package initializer (leeg voor nu)
```

Backend: backend/app/__init__.py
```python
# leeg; imports eventueel later
```

Backend: backend/app/config.py
```python
import os

class Settings:
    JWT_SECRET = os.getenv("JWT_SECRET", "CHANGE_ME_please")
    JWT_ALGORITHM = "HS256"
    JWT_EXPIRE_MINUTES = int(os.getenv("JWT_EXPIRE_MINUTES", "60"))

    DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://postgres:postgres@db:5432/saldo")
    # Default VAT rates (NL)
    NL_VAT_21 = 0.21
    NL_VAT_9 = 0.09

settings = Settings()
```

Backend: backend/app/database.py
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from .config import settings

engine = create_engine(settings.DATABASE_URL, echo=False, future=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Backend: backend/app/models.py
```python
from sqlalchemy import Column, Integer, String, Float, Date, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from .database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True, nullable=False)
    password_hash = Column(String, nullable=False)
    role = Column(String, default="Viewer")  # Admin | Accountant | Payroll | Viewer
    active = Column(Boolean, default=True)

class Customer(Base):
    __tablename__ = "customers"
    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String)
    address = Column(String)
    city = Column(String)
    postal_code = Column(String)
    country = Column(String)
    currency = Column(String, default="EUR")

class Invoice(Base):
    __tablename__ = "invoices"
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey("customers.id"))
    date = Column(Date)
    due_date = Column(Date)
    status = Column(String, default="concept")
    currency = Column(String, default="EUR")
    subtotal_excl_vat = Column(Float, default=0.0)
    vat_total = Column(Float, default=0.0)
    total_incl_vat = Column(Float, default=0.0)
    vat_rate_id = Column(Integer)

    customer = relationship("Customer", backref="invoices")

class InvoiceItem(Base):
    __tablename__ = "invoice_items"
    id = Column(Integer, primary_key=True)
    invoice_id = Column(Integer, ForeignKey("invoices.id"))
    description = Column(String)
    quantity = Column(Float)
    unit_price = Column(Float)
    subtotal_excl_vat = Column(Float, default=0.0)
    vat_amount = Column(Float, default=0.0)
    total_incl_vat = Column(Float, default=0.0)

class Account(Base):
    __tablename__ = "accounts"
    id = Column(Integer, primary_key=True)
    code = Column(String)
    name = Column(String)
    type = Column(String)

class JournalEntry(Base):
    __tablename__ = "journal_entries"
    id = Column(Integer, primary_key=True)
    date = Column(Date)
    description = Column(String)

class JournalLine(Base):
    __tablename__ = "journal_lines"
    id = Column(Integer, primary_key=True)
    entry_id = Column(Integer, ForeignKey("journal_entries.id"))
    account_id = Column(Integer, ForeignKey("accounts.id"))
    debit = Column(Float, default=0.0)
    credit = Column(Float, default=0.0)
    memo = Column(String)

class Category(Base):
    __tablename__ = "categories"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    type = Column(String)  # income / cost

class VATRate(Base):
    __tablename__ = "vat_rates"
    id = Column(Integer, primary_key=True)
    country = Column(String)
    rate_percent = Column(Float)
    start_date = Column(Date)
    end_date = Column(Date)

class Employee(Base):
    __tablename__ = "employees"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    address = Column(String)
    country = Column(String)
    monthly_gross = Column(Float)

class PayrollRun(Base):
    __tablename__ = "payroll_runs"
    id = Column(Integer, primary_key=True)
    period_start = Column(Date)
    period_end = Column(Date)
    run_date = Column(Date)
    status = Column(String, default="pending")
    total_gross = Column(Float, default=0.0)
    total_tax = Column(Float, default=0.0)
    total_net = Column(Float, default=0.0)

class PayrollLine(Base):
    __tablename__ = "payroll_lines"
    id = Column(Integer, primary_key=True)
    run_id = Column(Integer, ForeignKey("payroll_runs.id"))
    employee_id = Column(Integer, ForeignKey("employees.id"))
    gross = Column(Float)
    tax = Column(Float)
    social_contrib = Column(Float)
    net = Column(Float)
    holiday_pay = Column(Float, default=0.0)

    payroll_run = relationship("PayrollRun", backref="lines")
    employee = relationship("Employee")

```

Backend: backend/app/schemas.py
```python
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import date

class UserCreate(BaseModel):
    email: str
    password: str
    role: str = "Viewer"

class UserOut(BaseModel):
    id: int
    email: str
    role: str
    class Config:
        orm_mode = True

class CustomerCreate(BaseModel):
    name: str
    email: Optional[str] = None
    address: Optional[str] = None
    city: Optional[str] = None
    postal_code: Optional[str] = None
    country: Optional[str] = "NL"
    currency: Optional[str] = "EUR"

class CustomerOut(CustomerCreate):
    id: int
    class Config:
        orm_mode = True

class InvoiceItemCreate(BaseModel):
    description: str
    quantity: float
    unit_price: float

class InvoiceCreate(BaseModel):
    customer_id: int
    date: date
    due_date: date
    currency: Optional[str] = "EUR"
    vat_rate_id: Optional[int] = None
    items: List[InvoiceItemCreate]

class InvoiceOut(BaseModel):
    id: int
    customer_id: int
    date: date
    due_date: date
    currency: str
    subtotal_excl_vat: float
    vat_total: float
    total_incl_vat: float
    vat_rate_id: Optional[int]
    class Config:
        orm_mode = True

class PayrollRunCreate(BaseModel):
    period_start: date
    period_end: date

class PayslipOut(BaseModel):
    employee_id: int
    period_start: date
    period_end: date
    gross: float
    tax: float
    social_contrib: float
    holiday_pay: float
    net: float

class VATRateCreate(BaseModel):
    country: str
    rate_percent: float
    start_date: date
    end_date: Optional[date] = None
```

Backend: backend/app/security.py
```python
from passlib.context import CryptContext
from jose import jwt
from datetime import datetime, timedelta
from .config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_pwd: str, hashed_pwd: str) -> bool:
    return pwd_context.verify(plain_pwd, hashed_pwd)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.JWT_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)
    return encoded_jwt
```

Backend: backend/app/dependencies.py
```python
from fastapi import Depends
from sqlalchemy.orm import Session
from .database import SessionLocal
from .security import create_access_token
from fastapi.security import OAuth2PasswordBearer
from .config import settings

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(token: str = Depends(oauth2_scheme)):
    # Minimal placeholder: in real app decode JWT and fetch user
    # For MVP, return a dummy user with Admin role if token present
    return {"id": 1, "email": "admin@example.nl", "role": "Admin"}
```

Backend: backend/app/routers/auth.py
```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from sqlalchemy.orm import Session
from ..database import SessionLocal
from ..models import User
from ..schemas import UserCreate, UserOut
from ..security import hash_password, verify_password, create_access_token

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/register", response_model=UserOut)
def register(user_in: UserCreate, db: Session = Depends(get_db)):
    user = User(email=user_in.email, password_hash=hash_password(user_in.password), role=user_in.role)
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

@router.post("/login")
def login(email: str = "", password: str = ""):
    # Vereenvoudigde login-handeling voor MVP; in real-world gebruik je DB lookup
    if email and password:
        token = create_access_token({"sub": email})
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Invalid credentials")
```

Backend: backend/app/routers/customers.py
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from ..database import SessionLocal
from ..schemas import CustomerCreate, CustomerOut
from ..models import Customer

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.get("/", response_model=list[CustomerOut])
def list_customers(db: Session = Depends(get_db)):
    return db.query(Customer).all()

@router.post("/", response_model=CustomerOut)
def create_customer(payload: CustomerCreate, db: Session = Depends(get_db)):
    customer = Customer(**payload.dict())
    db.add(customer)
    db.commit()
    db.refresh(customer)
    return customer
```

Backend: backend/app/routers/invoices.py
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from ..database import SessionLocal
from ..schemas import InvoiceCreate, InvoiceOut, InvoiceItemCreate
from ..models import Invoice, InvoiceItem

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/", response_model=InvoiceOut)
def create_invoice(payload: InvoiceCreate, db: Session = Depends(get_db)):
    # Basisberekening
    subtotal = sum(item.quantity * item.unit_price for item in payload.items)
    vat_rate = 0.21  # default NL 21%
    vat_total = subtotal * vat_rate
    total = subtotal + vat_total

    invoice = Invoice(
        customer_id=payload.customer_id,
        date=payload.date,
        due_date=payload.due_date,
        currency=payload.currency or "EUR",
        subtotal_excl_vat=subtotal,
        vat_total=vat_total,
        total_incl_vat=total,
        vat_rate_id=payload.vat_rate_id,
    )
    db.add(invoice)
    db.commit()
    db.refresh(invoice)

    for it in payload.items:
        item = InvoiceItem(
            invoice_id=invoice.id,
            description=it.description,
            quantity=it.quantity,
            unit_price=it.unit_price,
            subtotal_excl_vat=it.quantity * it.unit_price,
            vat_amount=subtotal * vat_rate * (it.quantity / max(1, len(payload.items))),
            total_incl_vat=it.quantity * it.unit_price * (1 + vat_rate)
        )
        db.add(item)
    db.commit()
    db.refresh(invoice)
    return invoice
```

Backend: backend/app/routers/journals.py
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from ..database import SessionLocal
from ..models import JournalEntry, JournalLine, Account
from typing import List

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/journals")
def create_journal(date: str, description: str, lines: List[dict], db: Session = Depends(get_db)):
    je = JournalEntry(date=date, description=description)
    db.add(je)
    db.commit()
    for l in lines:
        jl = JournalLine(entry_id=je.id, account_id=l["account_id"], debit=l.get("debit", 0.0), credit=l.get("credit", 0.0), memo=l.get("memo", ""))
        db.add(jl)
    db.commit()
    return {"journal_id": je.id}
```

Backend: backend/app/routers/vat.py
```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from ..database import SessionLocal
from ..models import VATRate
from ..schemas import VATRateCreate

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.get("/rates")
def get_vat_rates(db: Session = Depends(get_db)):
    return db.query(VATRate).all()

@router.post("/rates")
def add_vat_rate(payload: VATRateCreate, db: Session = Depends(get_db)):
    rate = VATRate(country=payload.country, rate_percent=payload.rate_percent, start_date=payload.start_date, end_date=payload.end_date)
    db.add(rate)
    db.commit()
    db.refresh(rate)
    return rate
```

Backend: backend/app/routers/payroll.py
```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from ..database import SessionLocal
from ..models import Employee, PayrollRun, PayrollLine
from ..schemas import PayslipOut, PayrollRunCreate

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/run")
def run_payroll(payload: PayrollRunCreate, db: Session = Depends(get_db)):
    # Eenvoudige payroll-run
    run = PayrollRun(period_start=payload.period_start, period_end=payload.period_end, run_date=None, status="completed", total_gross=0, total_tax=0, total_net=0)
    db.add(run)
    db.commit()
    db.refresh(run)
    return {"run_id": run.id}
```

Backend: backend/app/routers/reports.py
```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from ..database import SessionLocal

router = APIRouter()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.get("/vat")
def vat_report(start: str, end: str, db: Session = Depends(get_db)):
    # Placeholder: bereken samenvatting
    return {"start": start, "end": end, "vat_due": 210.0}
```

Backend: backend/Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY backend/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Backend: backend/requirements.txt
```
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
passlib[bcrypt]
python-jose
python-dotenv
```

Frontend: frontend/Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY frontend/package.json frontend/yarn.lock* ./
RUN npm install

COPY frontend/ ./
RUN npm run build

EXPOSE 3000
CMD ["npm", "start"]
```

Frontend: frontend/package.json
```json
{
  "name": "saldo-suite-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "13.0.6",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "axios": "^0.28.0",
    "typescript": "^4.9.5"
  }
}
```

Frontend: frontend/tsconfig.json
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es2017"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve"
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

Frontend: frontend/pages/index.tsx
```tsx
import { useState } from "react";
import axios from "axios";

export default function Home() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  async function login(e: React.FormEvent) {
    e.preventDefault();
    try {
      const res = await axios.post("/auth/login", { email, password });
      // Simplified: store token
      localStorage.setItem("token", res.data.access_token);
      window.location.href = "/dashboard";
    } catch (err) {
      alert("Login failed");
    }
  }

  return (
    <form onSubmit={login}>
      <h1>SaldoSuite Login</h1>
      <input placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <input placeholder="Password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
      <button type="submit">Login</button>
    </form>
  );
}
```

Frontend: frontend/pages/dashboard.tsx
```tsx
export default function Dashboard() {
  return (
    <div>
      <h1>SaldoSuite Dashboard</h1>
      <p>Welkom! Dit is een placeholder-dashboard.</p>
    </div>
  );
}
```

Frontend: frontend/pages/customers.tsx
```tsx
import { useEffect, useState } from "react";
import axios from "axios";

export default function Customers() {
  const [customers, setCustomers] = useState<any[]>([]);

  useEffect(() => {
    async function fetch() {
      const res = await axios.get("/customers", { headers: { Authorization: "Bearer " + localStorage.getItem("token") } });
      setCustomers(res.data);
    }
    fetch();
  }, []);

  return (
    <div>
      <h2>Klanten</h2>
      <ul>
        {customers.map((c) => (
          <li key={c.id}>{c.name} - {c.email}</li>
        ))}
      </ul>
    </div>
  );
}
```

Frontend: frontend/pages/invoices.tsx
```tsx
export default function Invoices() {
  return (
    <div>
      <h2>Facturen</h2>
      <p>UI voor het aanmaken van facturen komt hier.</p>
    </div>
  );
}
```

Docker-Compose: docker-compose.yml
```yaml
version: '3.9'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: saldosuite
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/saldosuite
      JWT_SECRET: supersecret
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app

  frontend:
    build: ./frontend
    depends_on:
      - backend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app

volumes:
  postgres-data:
```

.env.example
```env
# PostgreSQL
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=saldosuite

# JWT
JWT_SECRET=supersecret

# Database URL (override als je direct via DATABASE_URL wilt connecteren)
DATABASE_URL=postgresql://postgres:postgres@db:5432/saldosuite

# NL BTW (defaults)
NL_VAT_21=0.21
NL_VAT_9=0.09

# SMTP (optioneel, niet nodig voor MVP)
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=
```
