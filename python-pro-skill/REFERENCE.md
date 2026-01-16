# Python Pro - Technical Reference

## Repository Pattern with Async SQLAlchemy

**When to use:** Clean separation of data access from business logic

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select
from typing import Generic, TypeVar, List, Optional
from abc import ABC, abstractmethod

T = TypeVar('T')

class BaseRepository(ABC, Generic[T]):
    """Abstract base repository"""
    
    def __init__(self, session: AsyncSession):
        self.session = session
    
    @abstractmethod
    async def get(self, id: int) -> Optional[T]:
        """Get entity by ID"""
        pass
    
    @abstractmethod
    async def list(self, skip: int = 0, limit: int = 100) -> List[T]:
        """List entities with pagination"""
        pass
    
    @abstractmethod
    async def create(self, entity: T) -> T:
        """Create new entity"""
        pass
    
    @abstractmethod
    async def update(self, id: int, entity: T) -> Optional[T]:
        """Update existing entity"""
        pass
    
    @abstractmethod
    async def delete(self, id: int) -> bool:
        """Delete entity by ID"""
        pass


class UserRepository(BaseRepository[User]):
    async def get(self, id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == id)
        )
        return result.scalar_one_or_none()
    
    async def list(self, skip: int = 0, limit: int = 100) -> List[User]:
        result = await self.session.execute(
            select(User).offset(skip).limit(limit)
        )
        return result.scalars().all()
    
    async def create(self, user: User) -> User:
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user
    
    async def update(self, id: int, user_data: dict) -> Optional[User]:
        user = await self.get(id)
        if not user:
            return None
        
        for key, value in user_data.items():
            setattr(user, key, value)
        
        await self.session.commit()
        await self.session.refresh(user)
        return user
    
    async def delete(self, id: int) -> bool:
        user = await self.get(id)
        if not user:
            return False
        
        await self.session.delete(user)
        await self.session.commit()
        return True


# Database session dependency
async def get_db() -> AsyncSession:
    async with SessionLocal() as session:
        yield session
```

---

## Background Tasks with Celery + FastAPI

**When to use:** Long-running tasks, async job processing

```python
from celery import Celery
from fastapi import BackgroundTasks, UploadFile
import asyncio

celery_app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

@celery_app.task
def process_video(video_id: int) -> dict:
    """Long-running task (minutes to hours)"""
    # Transcode video, generate thumbnails, etc.
    return {"video_id": video_id, "status": "completed"}


# FastAPI background task (short-lived, <60 seconds)
async def send_email(email: str, subject: str, body: str):
    """Quick background task (seconds)"""
    async with aiosmtplib.SMTP() as smtp:
        await smtp.send_message(...)


@app.post("/videos/upload")
async def upload_video(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    # Save file
    video_id = save_video(file)
    
    # Queue long-running task
    process_video.delay(video_id)
    
    # Queue quick background task
    background_tasks.add_task(
        send_email,
        "user@example.com",
        "Upload Successful",
        f"Video {video_id} uploaded"
    )
    
    return {"video_id": video_id, "status": "processing"}


# Check task status
@app.get("/videos/{video_id}/status")
async def get_video_status(video_id: int):
    task = process_video.AsyncResult(str(video_id))
    return {
        "status": task.state,
        "result": task.result if task.ready() else None
    }
```

---

## FastAPI with Advanced Type Safety

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel, Field, validator
from typing import Optional, List
import jwt
from datetime import datetime, timedelta

app = FastAPI(title="User Management API", version="1.0.0")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Pydantic models with validation
class UserCreate(BaseModel):
    email: str = Field(..., pattern=r'^[^@]+@[^@]+\.[^@]+$')
    password: str = Field(..., min_length=8)
    name: Optional[str] = None
    
    @validator('password')
    def validate_password(cls, v: str) -> str:
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain at least one digit')
        return v


class UserResponse(BaseModel):
    id: int
    email: str
    name: Optional[str]
    created_at: datetime
    
    class Config:
        from_attributes = True  # Pydantic v2


class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int


# Dependency injection with type safety
async def get_current_user(
    token: str = Depends(oauth2_scheme)
) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Could not validate credentials"
            )
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials"
        )
    
    user = await get_user_by_id(user_id)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user


# API endpoints with comprehensive error handling
@app.post("/users/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db)
) -> UserResponse:
    # Check if user already exists
    existing_user = await get_user_by_email(user_data.email, db)
    if existing_user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Email already registered"
        )
    
    # Create user with hashed password
    hashed_password = hash_password(user_data.password)
    user = User(
        email=user_data.email,
        password_hash=hashed_password,
        name=user_data.name
    )
    
    db.add(user)
    await db.commit()
    await db.refresh(user)
    
    return UserResponse.model_validate(user)


@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = UserRepository(db)
    user = await repo.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

---

## Async Semaphore for Concurrency Control

```python
import asyncio
from typing import List

async def fetch_url(session: aiohttp.ClientSession, url: str, semaphore: asyncio.Semaphore) -> str:
    async with semaphore:  # Limit concurrent requests
        async with session.get(url) as response:
            return await response.text()


async def fetch_all_urls(urls: List[str], max_concurrent: int = 10) -> List[str]:
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url, semaphore) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    return [r for r in results if not isinstance(r, Exception)]
```

---

## Custom Exception Hierarchy

```python
class AppError(Exception):
    """Base exception for application errors"""
    def __init__(self, message: str, code: str = "APP_ERROR"):
        self.message = message
        self.code = code
        super().__init__(message)


class ValidationError(AppError):
    """Raised when input validation fails"""
    def __init__(self, message: str, field: str):
        self.field = field
        super().__init__(message, code="VALIDATION_ERROR")


class NotFoundError(AppError):
    """Raised when resource is not found"""
    def __init__(self, resource: str, identifier: str):
        message = f"{resource} with id '{identifier}' not found"
        super().__init__(message, code="NOT_FOUND")


class AuthorizationError(AppError):
    """Raised when user lacks permission"""
    def __init__(self, action: str, resource: str):
        message = f"Not authorized to {action} {resource}"
        super().__init__(message, code="UNAUTHORIZED")


# FastAPI exception handler
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    status_map = {
        "VALIDATION_ERROR": 400,
        "NOT_FOUND": 404,
        "UNAUTHORIZED": 403,
    }
    return JSONResponse(
        status_code=status_map.get(exc.code, 500),
        content={"error": exc.code, "message": exc.message}
    )
```

---

## Performance Profiling

```python
import cProfile
import pstats
from functools import wraps
from typing import Callable, TypeVar
import time

T = TypeVar('T')

def profile(func: Callable[..., T]) -> Callable[..., T]:
    """Decorator to profile function execution"""
    @wraps(func)
    def wrapper(*args, **kwargs) -> T:
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(10)
        
        return result
    return wrapper


def timed(func: Callable[..., T]) -> Callable[..., T]:
    """Simple timing decorator"""
    @wraps(func)
    async def async_wrapper(*args, **kwargs) -> T:
        start = time.perf_counter()
        result = await func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    
    @wraps(func)
    def sync_wrapper(*args, **kwargs) -> T:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    
    if asyncio.iscoroutinefunction(func):
        return async_wrapper
    return sync_wrapper
```
