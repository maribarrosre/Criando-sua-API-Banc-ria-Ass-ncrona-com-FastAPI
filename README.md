# Criando-sua-API-Banc-ria-Ass-ncrona-com-FastAPI
O Desafio Criando sua API Bancária Assíncrona com FastAPI é um projeto desenvolvido para curso de PYTON da plataforma da DIO.

from pathlib import Path

PROJECT = Path("api-bancaria-fastapi")

FILES = {
    "requirements.txt": """
fastapi>=0.115,<1.0
uvicorn[standard]>=0.30,<1.0
sqlalchemy[asyncio]>=2.0,<3.0
aiosqlite>=0.20,<1.0
pydantic-settings>=2.5,<3.0
PyJWT[crypto]>=2.9,<3.0
pwdlib[argon2]>=0.2,<1.0
python-multipart>=0.0.9
email-validator>=2.2,<3.0
pytest>=8.0,<9.0
httpx>=0.27,<1.0
""",

    ".env.example": """
DATABASE_URL=sqlite+aiosqlite:///./banco.db
JWT_SECRET_KEY=troque-por-uma-chave-segura
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
""",

    ".gitignore": """
__pycache__/
*.py[cod]
.pytest_cache/
.venv/
venv/
.env
*.db
*.sqlite
.vscode/
.idea/
""",

    "app/__init__.py": "",

    "app/routers/__init__.py": "",

    "app/config.py": """
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str = "sqlite+aiosqlite:///./banco.db"
    jwt_secret_key: str = "chave-insegura-apenas-desenvolvimento"
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    model_config = SettingsConfigDict(
        env_file=".env",
        extra="ignore"
    )

settings = Settings()
""",

    "app/database.py": """
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine
)
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

class Base(DeclarativeBase):
    pass

engine = create_async_engine(
    settings.database_url,
    echo=False
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

async def create_tables():
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.create_all)
""",

    "app/models.py": """
from datetime import datetime, timezone
from decimal import Decimal
from sqlalchemy import DateTime, ForeignKey, Numeric, String
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.database import Base

def utc_now():
    return datetime.now(timezone.utc)

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    name: Mapped[str] = mapped_column(String(120))
    email: Mapped[str] = mapped_column(
        String(255), unique=True, index=True
    )
    hashed_password: Mapped[str] = mapped_column(String(255))

    account: Mapped["Account"] = relationship(
        back_populates="owner", uselist=False
    )

class Account(Base):
    __tablename__ = "accounts"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    number: Mapped[str] = mapped_column(
        String(20), unique=True, index=True
    )
    balance: Mapped[Decimal] = mapped_column(
        Numeric(14, 2), default=Decimal("0.00")
    )
    owner_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"), unique=True
    )

    owner: Mapped[User] = relationship(back_populates="account")
    transactions: Mapped[list["Transaction"]] = relationship(
        back_populates="account"
    )

class Transaction(Base):
    __tablename__ = "transactions"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    type: Mapped[str] = mapped_column(String(10))
    amount: Mapped[Decimal] = mapped_column(Numeric(14, 2))
    description: Mapped[str | None] = mapped_column(
        String(255), nullable=True
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utc_now
    )
    account_id: Mapped[int] = mapped_column(
        ForeignKey("accounts.id"), index=True
    )

    account: Mapped[Account] = relationship(
        back_populates="transactions"
    )
""",

    "app/schemas.py": """
from datetime import datetime
from decimal import Decimal
from pydantic import BaseModel, ConfigDict, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=2, max_length=120)
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)

class UserOut(BaseModel):
    id: int
    name: str
    email: EmailStr
    model_config = ConfigDict(from_attributes=True)

class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"

class TransactionCreate(BaseModel):
    amount: Decimal = Field(
        gt=0, max_digits=12, decimal_places=2
    )
    description: str | None = Field(
        default=None, max_length=255
    )

class TransactionOut(BaseModel):
    id: int
    type: str
    amount: Decimal
    description: str | None
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)

class AccountOut(BaseModel):
    id: int
    number: str
    balance: Decimal
    model_config = ConfigDict(from_attributes=True)

class StatementOut(BaseModel):
    account_number: str
    balance: Decimal
    transactions: list[TransactionOut]
""",

    "app/security.py": """
from datetime import datetime, timedelta, timezone
import jwt
from pwdlib import PasswordHash
from app.config import settings

password_hash = PasswordHash.recommended()

def hash_password(password: str) -> str:
    return password_hash.hash(password)

def verify_password(password: str, hashed_password: str) -> bool:
    return password_hash.verify(password, hashed_password)

def create_access_token(subject: str) -> str:
    expires = datetime.now(timezone.utc) + timedelta(
        minutes=settings.access_token_expire_minutes
    )
    payload = {"sub": subject, "exp": expires}
    return jwt.encode(
        payload,
        settings.jwt_secret_key,
        algorithm=settings.jwt_algorithm
    )

def decode_access_token(token: str) -> dict:
    return jwt.decode(
        token,
        settings.jwt_secret_key,
        algorithms=[settings.jwt_algorithm],
        options={"require": ["exp", "sub"]}
    )
""",

    "app/dependencies.py": """
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.models import User
from app.security import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:

    error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token inválido ou expirado.",
        headers={"WWW-Authenticate": "Bearer"}
    )

    try:
        payload = decode_access_token(token)
        user_id = int(payload["sub"])
    except (jwt.InvalidTokenError, KeyError, TypeError, ValueError):
        raise error

    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = result.scalar_one_or_none()

    if user is None:
        raise error

    return user
""",

    "app/routers/auth.py": """
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.models import Account, User
from app.schemas import Token, UserCreate, UserOut
from app.security import (
    create_access_token,
    hash_password,
    verify_password
)

router = APIRouter(prefix="/auth", tags=["Autenticação"])

@router.post(
    "/register",
    response_model=UserOut,
    status_code=status.HTTP_201_CREATED
)
async def register(
    data: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(
        select(User).where(User.email == data.email)
    )

    if result.scalar_one_or_none():
        raise HTTPException(
            status_code=409,
            detail="E-mail já cadastrado."
        )

    user = User(
        name=data.name,
        email=data.email,
        hashed_password=hash_password(data.password)
    )

    db.add(user)
    await db.flush()

    account = Account(
        number=f"{user.id:08d}",
        balance=0,
        owner_id=user.id
    )

    db.add(account)
    await db.commit()
    await db.refresh(user)

    return user

@router.post("/login", response_model=Token)
async def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(
        select(User).where(User.email == form.username)
    )

    user = result.scalar_one_or_none()

    if user is None or not verify_password(
        form.password, user.hashed_password
    ):
        raise HTTPException(
            status_code=401,
            detail="E-mail ou senha incorretos.",
            headers={"WWW-Authenticate": "Bearer"}
        )

    return Token(
        access_token=create_access_token(str(user.id))
    )
""",

    "app/routers/accounts.py": """
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.dependencies import get_current_user
from app.models import Account, User
from app.schemas import AccountOut

router = APIRouter(prefix="/accounts", tags=["Contas"])

@router.get("/me", response_model=AccountOut)
async def get_my_account(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(
        select(Account).where(
            Account.owner_id == current_user.id
        )
    )

    account = result.scalar_one_or_none()

    if account is None:
        raise HTTPException(
            status_code=404,
            detail="Conta não encontrada."
        )

    return account
""",

    "app/routers/transactions.py": """
from decimal import Decimal
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.dependencies import get_current_user
from app.models import Account, Transaction, User
from app.schemas import (
    StatementOut,
    TransactionCreate,
    TransactionOut
)

router = APIRouter(
    prefix="/transactions",
    tags=["Transações"]
)

async def get_user_account(
    db: AsyncSession,
    user_id: int
) -> Account:
    result = await db.execute(
        select(Account).where(Account.owner_id == user_id)
    )
    account = result.scalar_one_or_none()

    if account is None:
        raise HTTPException(
            status_code=404,
            detail="Conta não encontrada."
        )

    return account

@router.post(
    "/deposit",
    response_model=TransactionOut,
    status_code=status.HTTP_201_CREATED
)
async def deposit(
    data: TransactionCreate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    account = await get_user_account(db, current_user.id)

    transaction = Transaction(
        type="DEPOSIT",
        amount=data.amount,
        description=data.description,
        account_id=account.id
    )

    account.balance = Decimal(account.balance) + data.amount
    db.add(transaction)

    await db.commit()
    await db.refresh(transaction)

    return transaction

@router.post(
    "/withdraw",
    response_model=TransactionOut,
    status_code=status.HTTP_201_CREATED
)
async def withdraw(
    data: TransactionCreate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    account = await get_user_account(db, current_user.id)

    if Decimal(account.balance) < data.amount:
        raise HTTPException(
            status_code=400,
            detail="Saldo insuficiente para realizar o saque."
        )

    transaction = Transaction(
        type="WITHDRAWAL",
        amount=data.amount,
        description=data.description,
        account_id=account.id
    )

    account.balance = Decimal(account.balance) - data.amount
    db.add(transaction)

    await db.commit()
    await db.refresh(transaction)

    return transaction

@router.get("/statement", response_model=StatementOut)
async def statement(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    account = await get_user_account(db, current_user.id)

    result = await db.execute(
        select(Transaction)
        .where(Transaction.account_id == account.id)
        .order_by(
            Transaction.created_at.desc(),
            Transaction.id.desc()
        )
    )

    transactions = result.scalars().all()

    return StatementOut(
        account_number=account.number,
        balance=account.balance,
        transactions=transactions
    )
""",

    "app/main.py": """
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.database import create_tables
from app.routers import auth, accounts, transactions

@asynccontextmanager
async def lifespan(app: FastAPI):
    await create_tables()
    yield

app = FastAPI(
    title="API Bancária Assíncrona",
    description="API REST para contas, depósitos, saques e extratos.",
    version="1.0.0",
    lifespan=lifespan
)

app.include_router(auth.router)
app.include_router(accounts.router)
app.include_router(transactions.router)

@app.get("/", tags=["Início"])
async def root():
    return {
        "message": "API Bancária está funcionando.",
        "documentation": "/docs"
    }
""",

    "tests/test_security.py": """
from app.security import (
    hash_password,
    verify_password,
    create_access_token,
    decode_access_token
)

def test_password_hash():
    password = "SenhaSegura123"
    hashed = hash_password(password)

    assert hashed != password
    assert verify_password(password, hashed)
    assert not verify_password("senha-errada", hashed)

def test_jwt_token():
    token = create_access_token("123")
    payload = decode_access_token(token)

    assert payload["sub"] == "123"
    assert "exp" in payload
""",

    "README.md": """
# API Bancária Assíncrona com FastAPI

Projeto educacional de desenvolvimento Backend com Python.

## Funcionalidades

- Cadastro de usuários.
- Criação automática de contas.
- Autenticação JWT.
- Hash de senhas com Argon2.
- Consulta de conta.
- Depósitos e saques.
- Extrato bancário.
- Validação de saldo.
- Proteção de rotas.

## Tecnologias

- Python
- FastAPI
- SQLAlchemy assíncrono
- SQLite
- Pydantic
- JWT
- Argon2
- Pytest

## Instalação

Crie um ambiente virtual:

    python -m venv .venv

Ative no Windows:

    .venv\\Scripts\\activate

Instale as dependências:

    pip install -r requirements.txt

Copie `.env.example` para `.env` e configure uma chave JWT segura.

Execute:

    uvicorn app.main:app --reload

Acesse a documentação:

    http://127.0.0.1:8000/docs

## Endpoints

| Método | Rota | Descrição |
|---|---|---|
| POST | /auth/register | Cadastro |
| POST | /auth/login | Login JWT |
| GET | /accounts/me | Consultar conta |
| POST | /transactions/deposit | Depósito |
| POST | /transactions/withdraw | Saque |
| GET | /transactions/statement | Extrato |

## Testes

    pytest

## Observação

Projeto para fins educacionais. Não utilizar como sistema bancário real sem auditoria e melhorias adicionais de segurança.
""",

    "LICENSE": """
MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.
"""
}


def create_project():
    for filename, content in FILES.items():
        path = PROJECT / filename
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(content.strip() + "\n", encoding="utf-8")
        print(f"Arquivo criado: {path}")

    print("\nProjeto criado com sucesso!")
    print(f"Pasta: {PROJECT.resolve()}")


if __name__ == "__main__":
    create_project()
