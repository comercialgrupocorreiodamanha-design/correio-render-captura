import os
import base64
from datetime import datetime

from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from playwright.sync_api import sync_playwright
from email.message import EmailMessage


URL = "https://www.correiodamanha.com.br"
DRIVE_FOLDER_ID = "1bkp5Jz6hiP_KYFBPBX1cf_2zhQcGsic4"
EMAIL_DESTINO = "patrick@correiodamanha.net.br"
EMAIL_ORIGEM = "no-reply@correiodamanha.net.br"

SERVICE_ACCOUNT_FILE = "service-account.json"


def captura_desktop(playwright):
    browser = playwright.chromium.launch()
    page = browser.new_page(viewport={"width": 1920, "height": 1080})
    page.goto(URL, wait_until="networkidle")
    filename = f"Correio_Desktop_{datetime.now().strftime('%Y-%m-%d_%H-%M-%S')}.png"
    page.screenshot(path=filename, full_page=True)
    browser.close()
    return filename


def captura_mobile(playwright):
    iphone = playwright.devices["iPhone 13 Pro"]
    browser = playwright.chromium.launch()
    context = browser.new_context(**iphone)
    page = context.new_page()
    page.goto(URL, wait_until="networkidle")
    filename = f"Correio_Mobile_{datetime.now().strftime('%Y-%m-%d_%H-%M-%S')}.png"
    page.screenshot(path=filename, full_page=True)
    browser.close()
    return filename


def upload_drive(filepath):
    creds = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE,
        scopes=["https://www.googleapis.com/auth/drive"]
    )
    service = build("drive", "v3", credentials=creds)

    file_metadata = {
        "name": os.path.basename(filepath),
        "parents": [DRIVE_FOLDER_ID]
    }
    media = MediaFileUpload(filepath, mimetype="image/png")

    result = service.files().create(
        body=file_metadata,
        media_body=media,
        fields="id"
    ).execute()

    return result.get("id")


def enviar_email(filepaths):
    creds = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE,
        scopes=["https://www.googleapis.com/auth/gmail.send"]
    )
    service = build("gmail", "v1", credentials=creds)

    msg = EmailMessage()
    msg["To"] = EMAIL_DESTINO
    msg["From"] = EMAIL_ORIGEM
    msg["Subject"] = "Print Diário – Correio da Manhã"

    msg.set_content("Segue em anexo o print diário nas versões Desktop e Mobile.")

    for fp in filepaths:
        with open(fp, "rb") as f:
            data = f.read()
        msg.add_attachment(
            data,
            maintype="image",
            subtype="png",
            filename=os.path.basename(fp)
        )

    encoded = base64.urlsafe_b64encode(msg.as_bytes()).decode()

    service.users().messages().send(
        userId="me",
        body={"raw": encoded}
    ).execute()


def main():
    with sync_playwright() as p:
        desktop = captura_desktop(p)
        mobile = captura_mobile(p)

    upload_drive(desktop)
    upload_drive(mobile)
    enviar_email([desktop, mobile])

    print("✔ Capturas realizadas, enviadas e salvas no Drive.")


if __name__ == "__main__":
    main()
