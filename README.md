api_bancaria/
│
├── main.py              # Ponto de entrada da aplicação FastAPI
├── database.py          # Configuração da base de dados (ex: SQLAlchemy assíncrono)
├── models.py            # Modelos/Tabelas da Base de Dados (Conta e Transação)
├── schemas.py           # Schemas Pydantic para validação de dados (Input/Output)
├── auth.py              # Lógica de Autenticação e JWT
└── requirements.txt     # Dependências do projetofastapi>=0.100.0
uvicorn[standard]>=0.22.0
sqlalchemy>=2.0.0
aiosqlite>=0.19.0
pydantic>=2.0.0
passlib[bcrypt]>=1.7.4
python-jose[cryptography]>=3.3.0
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "sqlite+aiosqlite:///./banco.db"

engine = create_async_engine(DATABASE_URL, echo=True, future=True)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
  pass


async def get_db():
  async with async_session() as session:
    yield session
    from sqlalchemy import CheckConstraint, Float, ForeignKey, Integer, String
from sqlalchemy.orm import relationship, Mapped, mapped_column
from database import Base


class ContaModel(Base):
  __tablename__ = "contas"

  id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
  numero_conta: Mapped[str] = mapped_column(
      String, unique=True, index=True, nullable=False
  )
  senha_hash: Mapped[str] = mapped_column(String, nullable=False)
  saldo: Mapped[float] = mapped_column(Float, default=0.0, nullable=False)

  transacoes = relationship(
      "TransacaoModel", back_populates="conta", cascade="all, delete-orphan"
  )


class TransacaoModel(Base):
  __tablename__ = "transacoes"

  id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
  tipo: Mapped[str] = mapped_column(
      String, nullable=False
  )  # "DEPOSITO" ou "SAQUE"
  valor: Mapped[float] = mapped_column(Float, nullable=False)
  conta_id: Mapped[int] = mapped_column(
      Integer, ForeignKey("contas.id"), nullable=False
  )

  conta = relationship("ContaModel", back_populates="transacoes")
  from pydantic import BaseModel, Field


class ContaCreate(BaseModel):
  numero_conta: str
  senha: str


class Token(BaseModel):
  access_token: str
  token_type: str


class TransacaoCreate(BaseModel):
  tipo: str = Field(..., description="DEPOSITO ou SAQUE")
  valor: float = Field(..., gt=0, description="O valor deve ser positivo")


class TransacaoResponse(BaseModel):
  id: int
  tipo: str
  valor: float

  class Config:
    from_attributes = True
    from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

SECRET_KEY = "sua_chave_secreta_super_segura"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def verificar_senha(senha_pura, senha_hash):
  return pwd_context.verify(senha_pura, senha_hash)


def criar_hash_senha(senha):
  return pwd_context.hash(senha)


def criar_token_acesso(data: dict):
  to_encode = data.copy()
  expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
  to_encode.update({"exp": expire})
  return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
  from auth import criar_hash_senha, criar_token_acesso, verificar_senha
from database import engine, get_db, Base
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
import models
import schemas
from sqlalchemy.future import select
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI(
    title="API Bancária Assíncrona",
    description=(
        "API desenvolvida com FastAPI para gestão de contas, transações e JWT"
        "."
    ),
    version="1.0.0",
)

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


@app.on_event("startup")
async def startup():
  async with engine.begin() as conn:
    await conn.run_sync(Base.metadata.create_all)


# Rota para Criar Conta
@app.post("/contas/", status_code=status.HTTP_201_CREATED)
async def criar_conta(
    conta: schemas.ContaCreate, db: AsyncSession = Depends(get_db)
):
  result = await db.execute(
      select(models.ContaModel).filter_by(numero_conta=conta.numero_conta)
  )
  if result.scalars().first():
    raise HTTPException(status_code=400, detail="Conta já existe.")

  nova_conta = models.ContaModel(
      numero_conta=conta.numero_conta,
      senha_hash=criar_hash_senha(conta.senha),
      saldo=0.0,
  )
  db.add(nova_conta)
  await db.commit()
  return {"mensagem": "Conta criada com sucesso!"}


# Rota de Autenticação (Gera o Token JWT)
@app.post("/token", response_model=schemas.Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
  result = await db.execute(
      select(models.ContaModel).filter_by(numero_conta=form_data.username)
  )
  conta = result.scalars().first()
  if not conta or not verificar_senha(form_data.password, conta.senha_hash):
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Credenciais inválidas",
        headers={"WWW-Authenticate": "Bearer"},
    )

  access_token = criar_token_acesso(data={"sub": conta.numero_conta})
  return {"access_token": access_token, "token_type": "bearer"}


# Dependência para Obter o Utilizador Atual via JWT
async def obter_conta_atual(
    token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)
):
  credentials_exception = HTTPException(
      status_code=status.HTTP_401_UNAUTHORIZED,
      detail="Não foi possível validar as credenciais",
      headers={"WWW-Authenticate": "Bearer"},
  )
  try:
    payload = jwt.decode(
        token, auth_secret := "sua_chave_secreta_super_segura", algorithms=["HS256"]
    )
    numero_conta: str = payload.get("sub")
    if numero_conta is None:
      raise credentials_exception
  except JWTError:
    raise credentials_exception

  result = await db.execute(
      select(models.ContaModel).filter_by(numero_conta=numero_conta)
  )
  conta = result.scalars().first()
  if conta is None:
    raise credentials_exception
  return conta


# Cadastro de Transações (Depósito / Saque com validações)
@app.post("/transacoes/", response_model=schemas.TransacaoResponse)
async def criar_transacao(
    transacao: schemas.TransacaoCreate,
    conta_atual: models.ContaModel = Depends(obter_conta_atual),
    db: AsyncSession = Depends(get_db),
):
  if transacao.tipo not in ["DEPOSITO", "SAQUE"]:
    raise HTTPException(
        status_code=400, detail="Tipo de transação inválido. Use DEPOSITO ou SAQUE."
    )

  if transacao.valor <= 0:
    raise HTTPException(
        status_code=400, detail="O valor da transação deve ser positivo."
    )

  if transacao.tipo == "SAQUE":
    if conta_atual.saldo < transacao.valor:
      raise HTTPException(status_code=400, detail="Saldo insuficiente.")
    conta_atual.saldo -= transacao.valor
  elif transacao.tipo == "DEPOSITO":
    conta_atual.saldo += transacao.valor

  nova_transacao = models.TransacaoModel(
      tipo=transacao.tipo, valor=transacao.valor, conta_id=conta_atual.id
  )

  db.add(nova_transacao)
  await db.commit()
  await db.refresh(nova_transacao)
  return nova_transacao


# Exibição de Extrato
@app.get("/extrato/")
async def ver_extrato(
    conta_atual: models.ContaModel = Depends(obter_conta_atual),
    db: AsyncSession = Depends(get_db),
):
  result = await db.execute(
      select(models.TransacaoModel).filter_by(conta_id=conta_atual.id)
  )
  transacoes = result.scalars().all()
  return {
      "numero_conta": conta_atual.numero_conta,
      "saldo_atual": conta_atual.saldo,
      "transacoes": transacoes,
  }
  pip install -r requirements.txt
  uvicorn main:app --reload


