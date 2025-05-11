#main.py

def get_current_user(message: Annotated[Message, Depends(parse_message)]) -> User | None:
    if not message:
        log.info("get_current_user: no message to authenticate, skipping user lookup.")
        return None
    phone = message.from_
    log.info("get_current_user: authenticating phone '%s'", phone)
    user = message_service.authenticate_user_by_phone_number(phone)
    if user:
        log.info("get_current_user: user found %s", user)
    else:
        log.info("get_current_user: no user found for phone '%s'", phone)
    return user

    # Authorization check
    if not user:
        log.warning("receive_whatsapp: unauthorized access attempt, no user matched.")
        raise HTTPException(status_code=401, detail="Unauthorized")


#message_service.py
def authenticate_user_by_phone_number(phone_number: str) -> User | None:
    allowed_users = [
        {"id": 1, "phone": "+50660285276", "first_name": "John", "last_name": "Doe", "role": "basic"},
        {"id": 2, "phone": "50660285276", "first_name": "Jane", "last_name": "Smith", "role": "basic"}
    ]

    for user in allowed_users:
        if user["phone"] == phone_number:
            return User(**user)
    return None



    
