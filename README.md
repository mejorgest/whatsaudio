# whatsaudio
import logging
import time
import threading
from typing_extensions import Annotated
from dotenv import load_dotenv
import os
#main.py
def parse_audio_file(message: Annotated[Message, Depends(parse_message)]) -> Audio | None:
    if message and message.type == "audio":
        return message.audio
    return None
@app.post("/webhook", status_code=200)
async def receive_whatsapp(
        request: Request,
        user: Annotated[User, Depends(get_current_user)],
        user_message: Annotated[str, Depends(message_extractor)],
        image: Annotated[Image, Depends(parse_image_file)],
):
    # Debug logs: headers and payload
    headers = dict(request.headers)
    log.info("receive_whatsapp: headers=%s", headers)
    body_bytes = await request.body()
    try:
        payload = json.loads(body_bytes)
        log.info("receive_whatsapp: payload=%s", payload)
    except Exception as e:
        log.error("receive_whatsapp: error parsing JSON body: %s", e)
        log.debug("receive_whatsapp: raw body bytes=%s", body_bytes)

    # If there's no meaningful content, acknowledge
    if not user and not user_message and not image:
        log.info("receive_whatsapp: empty or status-only message, returning OK.")
        return {"status": "ok"}

    # Authorization check
    if not user:
        log.warning("receive_whatsapp: unauthorized access attempt, no user matched.")
        raise HTTPException(status_code=401, detail="Unauthorized")

    # Image handling
    if image:
        log.info("receive_whatsapp: image received, id=%s", image.id)
        return {"status": "ok"}

    # Text or audio message
    if user_message:
        log.info("receive_whatsapp: Received message from user %s %s (%s): %s",
                 user.first_name, user.last_name, user.phone, user_message)
        # Responder al mensaje usando el agente
        threading.Thread(target=message_service.respond_and_send_message, args=(user_message, user)).start()
    return {"status": "ok"}


    

def message_extractor(
        message: Annotated[Message, Depends(parse_message)],
        audio: Annotated[Audio, Depends(parse_audio_file)],
):
    if audio:
        return message_service.transcribe_audio(audio)
    if message and message.text:
        return message.text.body
    return None


#schema.py
from pydantic import BaseModel, Field

class Audio(BaseModel):
    mime_type: str
    sha256: str
    id: str
    voice: bool


#message_service.py
from typing import BinaryIO
from app.schema import Audio, User
from openai import OpenAI
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")
WHATSAPP_API_KEY = "EAAOqKukI2PMBO5ryv6ZBSOPj7Tw6YmlwJiZAnhZAdkcOoG7g3CtssSbXZB5AP9AFIXghkc88NHUd1hRl3DWKci31PZCGjTMhdFCX86v0r2qmDiew1ni63EX4YieEVntqIhAvZC2uukY29D4b5tcZAh5o3A4rj8dpoqzK34LltMk5AmBSYGwk3XW3svvIqlURsouJsaXKJFxZAwoCWgGZCL0AbUwuyjEChI3ipDBJsLkg7"

# Cliente OpenAI con la API key configurada
llm = OpenAI(api_key=OPENAI_API_KEY)

def transcribe_audio_file(audio_file: BinaryIO) -> str:
    if not audio_file:
        return "No audio file provided"
    try:
        transcription = llm.audio.transcriptions.create(
            file=audio_file,
            model="whisper-1",
            response_format="text"
        )
        return transcription
    except Exception as e:
        raise ValueError("Error transcribing audio") from e

def transcribe_audio(audio: Audio) -> str:
    file_path = download_file_from_facebook(audio.id, "audio", audio.mime_type)
    with open(file_path, 'rb') as audio_binary:
        transcription = transcribe_audio_file(audio_binary)
    try:
        os.remove(file_path)
    except Exception as e:
        print(f"Failed to delete file: {e}")
    return transcription


def download_file_from_facebook(file_id: str, file_type: str, mime_type: str) -> str | None:
    # First GET request to retrieve the download URL
    url = f"https://graph.facebook.com/v19.0/{file_id}"
    headers = {"Authorization": f"Bearer {WHATSAPP_API_KEY}"}
    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        download_url = response.json().get('url')

        # Second GET request to download the file
        response = requests.get(download_url, headers=headers)

        if response.status_code == 200:
            file_extension = mime_type.split('/')[-1].split(';')[0]  # Extract file extension from mime_type
            file_path = f"{file_id}.{file_extension}"
            with open(file_path, 'wb') as file:
                file.write(response.content)
            if file_type == "image" or file_type == "audio":
                return file_path

        raise ValueError(f"Failed to download file. Status code: {response.status_code}")
    raise ValueError(f"Failed to retrieve download URL. Status code: {response.status_code}")

