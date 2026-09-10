import os
import json
import hashlib
import requests
from pathlib import Path

TELEGRAM_TOKEN = os.environ["TELEGRAM_TOKEN"]

PDF_URL = (
    "https://aspazijasvsk.lv/"
    "macibu-process/izmainas-stundu-saraksta-ritdienai/"
)

USERS_FILE = Path("users.json")
STATE_FILE = Path("last_hash.txt")


def telegram(method, data=None, files=None):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/{method}"

    response = requests.post(
        url,
        data=data,
        files=files,
        timeout=60
    )

    response.raise_for_status()
    return response.json()


def load_users():
    if not USERS_FILE.exists():
        return []

    try:
        return json.loads(USERS_FILE.read_text())
    except Exception:
        return []


def save_users(users):
    USERS_FILE.write_text(
        json.dumps(users, ensure_ascii=False, indent=2)
    )


def check_new_users():
    users = load_users()

    offset_file = Path("telegram_offset.txt")

    if offset_file.exists():
        offset = int(offset_file.read_text())
    else:
        offset = None

    data = {}

    if offset is not None:
        data["offset"] = offset

    result = telegram("getUpdates", data)

    updates = result.get("result", [])

    for update in updates:
        update_id = update["update_id"]

        offset = update_id + 1

        message = update.get("message")

        if not message:
            continue

        chat = message.get("chat")

        if not chat:
            continue

        # Tikai privātās sarunas
        if chat.get("type") != "private":
            continue

        chat_id = chat["id"]
        text = message.get("text", "")

        if text.startswith("/start"):

            if chat_id not in users:
                users.append(chat_id)

                telegram(
                    "sendMessage",
                    {
                        "chat_id": chat_id,
                        "text": (
                            "👋 Sveiks!\n\n"
                            "Tu esi pieslēdzies "
                            "Jelgavas Aspazijas vidusskolas "
                            "stundu saraksta paziņojumiem.\n\n"
                            "📚 Kad būs publicēts jauns "
                            "stundu saraksts, tu saņemsi to šeit."
                        )
                    }
                )

        elif text.startswith("/stop"):

            if chat_id in users:
                users.remove(chat_id)

                telegram(
                    "sendMessage",
                    {
                        "chat_id": chat_id,
                        "text": (
                            "Paziņojumi ir izslēgti.\n\n"
                            "Ja vēlies tos atkal saņemt, "
                            "nosūti /start"
                        )
                    }
                )

    save_users(users)

    if offset is not None:
        offset_file.write_text(str(offset))


def download_file():
    response = requests.get(
        PDF_URL,
        timeout=60,
        headers={
            "User-Agent": "Mozilla/5.0"
        }
    )

    response.raise_for_status()

    data = response.content

    if not data.startswith(b"%PDF"):
        raise RuntimeError(
            "Norādītā saite neatgrieza PDF failu."
        )

    return data


def get_hash(data):
    return hashlib.sha256(data).hexdigest()


def get_old_hash():
    if not STATE_FILE.exists():
        return None

    return STATE_FILE.read_text().strip()


def save_hash(file_hash):
    STATE_FILE.write_text(file_hash)


def send_pdf(pdf_data):
    users = load_users()

    for chat_id in users:

        try:

            telegram(
                "sendDocument",
                data={
                    "chat_id": chat_id,
                    "caption": (
                        "📚 Jauns stundu saraksts!\n\n"
                        "Jelgavas Aspazijas vidusskola"
                    )
                },
                files={
                    "document": (
                        "stundu_saraksts.pdf",
                        pdf_data,
                        "application/pdf"
                    )
                }
            )

            print(
                f"Nosūtīts lietotājam {chat_id}"
            )

        except Exception as error:

            print(
                f"Neizdevās nosūtīt lietotājam "
                f"{chat_id}: {error}"
            )


def main():

    print("Pārbaudu stundu sarakstu...")

    # Vispirms pārbaudām jaunus /start
    check_new_users()

    # Lejupielādējam sarakstu
    pdf_data = download_file()

    new_hash = get_hash(pdf_data)
    old_hash = get_old_hash()

    print("Jaunais hash:", new_hash)
    print("Vecais hash:", old_hash)

    # Pirmā palaišana
    if old_hash is None:

        print(
            "Pirmā pārbaude. "
            "Saraksts netiek nosūtīts."
        )

        save_hash(new_hash)

        return

    # Nekas nav mainījies
    if new_hash == old_hash:

        print(
            "Stundu saraksts nav mainījies."
        )

        return

    # Ir jauns saraksts
    print(
        "ATRĀTS JAUNS STUNDU SARAKSTS!"
    )

    send_pdf(pdf_data)

    save_hash(new_hash)

    print(
        "Jaunais saraksts nosūtīts."
    )


if __name__ == "__main__":
    main()