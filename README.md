# python-backup.py
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from database import Base, engine, SessionLocal
from models import User, Task
import schemas, auth

Base.metadata.create_all(bind=engine)

app = FastAPI(title="Task Manager API")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/register")
def register(user: schemas.UserCreate, db: Session = Depends(get_db)):
    hashed = auth.hash_password(user.password)
    db_user = User(username=user.username, password=hashed)
    db.add(db_user)
    db.commit()
    return {"msg": "User registered"}

@app.post("/login")
def login(user: schemas.UserCreate, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.username == user.username).first()
    if not db_user or not auth.verify(user.password, db_user.password):
        raise HTTPException(401, "Invalid credentials")
    token = auth.create_token(db_user.username)
    return {"access_token": token}

@app.post("/tasks")
def create_task(task: schemas.TaskCreate, db: Session = Depends(get_db)):
    db_task = Task(title=task.title)
    db.add(db_task)
    db.commit()
    return {"msg": "Task created"}

@app.get("/tasks")
def list_tasks(db: Session = Depends(get_db)):
    return db.query(Task).all()

@app.put("/tasks/{task_id}")
def complete_task(task_id: int, db: Session = Depends(get_db)):
    task = db.query(Task).get(task_id)
    if not task:
        raise HTTPException(404, "Task not found")
    task.completed = True
    db.commit()
    return {"msg": "Task completed"}

@app.delete("/tasks/{task_id}")
def delete_task(task_id: int, db: Session = Depends(get_db)):
    task = db.query(Task).get(task_id)
    if not task:
        raise HTTPException(404, "Task not found")
    db.delete(task)
    db.commit()
    return {"msg": "Task deleted"}
