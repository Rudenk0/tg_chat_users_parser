# Собираем участников из закрытых чатов Telegram

[В предыдущей статье](https://vc.ru/dev/1480225-osint-v-rabote-it-rekrutera-avtomatiziruem-poisk-po-nikneimam) мы разобрали, как самостоятельно собирать никнеймы участников из открытых чатов. Однако часто бывает так, что эта информация скрыта, как, например, в чате DevOps специалистов [@devops_jobs](https://t.me/devops_jobs).  
Как видим на картинке ниже, участников много, но доступ к их списку закрыт администраторами.

<p align="center">
  <img src="https://github.com/user-attachments/assets/718c32b3-d8de-4314-ab25-dca3d806cfe1" alt="Пример скрытого чата" title="Пример скрытого чата" width="200">
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/cf01e098-d413-4336-a498-fc233c139e2f" alt="Список участников" title="Список участников" width="200">
</p>

Скрипт, который собирает данные через обращение к API Telegram, не сможет собрать участников из таких чатов. Поэтому мы будем собирать всех, кто когда-либо оставлял сообщения в чате. Таким методом нам не получится собрать всех участников чатов, но мы сможем собрать данные тех, кто, возможно, уже в нем не состоит.

## Установка необходимых библиотек

Для начала установим необходимые библиотеки:

```pip install telethon asyncio atexit```


Скопируйте ваши данные и запустите скрипт:


```import asyncio
import atexit
from telethon.sync import TelegramClient
from telethon.errors import FloodWaitError

# Укажите ваши API ID и API Hash
api_id = 'ВАШ API ID'
api_hash = 'ВАШ API HASH'
phone_number = 'ВАШ НОМЕР ТЕЛЕФОНА'

# Создание клиента
client = TelegramClient('session_name', api_id, api_hash)

# Переменная для указания чата
chat_to_extract = 'АДРЕС КАНАЛА БЕЗ СОБАЧКИ'  # Замените на нужный чат или username

# Коллекция юзернеймов
usernames = set()

# Функция для сохранения данных
def save_usernames(usernames, file_name):
    """Сохраняет уникальные юзернеймы в файл."""
    try:
        with open(file_name, 'w') as file:
            for username in sorted(usernames):  # Сортировка для удобства просмотра
                file.write(username + '\n')
        print(f"Usernames saved to {file_name}")
    except Exception as e:
        print(f"Error saving usernames: {e}")

# Функция для извлечения юзернеймов
async def fetch_usernames():
    await client.start(phone_number)

    # Получаем информацию о чате
    chat_entity = await client.get_entity(chat_to_extract)
    chat_name = chat_entity.username or chat_entity.title or "chat"
    file_name = f"{chat_name}_users.txt"

    sender_id_to_username = {}
    total_messages = 0

    try:
        async for message in client.iter_messages(chat_to_extract):
            total_messages += 1
            sender_id = message.sender_id
            if sender_id:
                if sender_id in sender_id_to_username:
                    username = sender_id_to_username[sender_id]
                else:
                    try:
                        sender = await client.get_entity(sender_id)
                    except FloodWaitError as e:
                        print(f"Rate limit exceeded. Waiting for {e.seconds} seconds...")
                        await asyncio.sleep(e.seconds)
                        sender = await client.get_entity(sender_id)
                    except Exception as e:
                        print(f"Failed to get entity for sender_id {sender_id}: {e}")
                        continue

                    username = sender.username.replace('@', '') if sender.username else None
                    sender_id_to_username[sender_id] = username

                if username and username not in usernames:
                    usernames.add(username)
                    print(f"Added new username: {username}")

                    # Сообщение и сохранение при кратности 100
                    if len(usernames) % 100 == 0:
                        save_usernames(usernames, file_name)
                        print(f"Collected {len(usernames)} unique usernames... Intermediate results saved.")

                elif username is None:
                    print(f"No username for sender_id {sender_id}")

    except KeyboardInterrupt:
        print("\nOperation interrupted manually. Saving progress...")
        save_usernames(usernames, file_name)
        print("Progress saved successfully.")
        raise

    except Exception as e:
        print(f"An error occurred: {e}")

    # Финальное сохранение данных
    save_usernames(usernames, file_name)
    print(f"All usernames have been saved successfully in {file_name}.")
    print(f"Total messages processed: {total_messages}")
    print(f"Total unique usernames collected: {len(usernames)}")

# Регистрируем функцию для выполнения при завершении
atexit.register(save_usernames, usernames, f'{chat_to_extract}_users.txt')

with client:
    client.loop.run_until_complete(fetch_usernames()) 
```

Данный скрипт собирает всех участников, кто оставил сообщение в чате, и сохраняет данные каждые 100 собранных никнеймов. Это позволяет эффективно собирать информацию даже из чатов, где скрыты участники.

<p align="center"> <img src="https://github.com/user-attachments/assets/6f7291a6-e0eb-4534-bc81-a1dda0bae873" alt="Пример работы программы" title="Пример работы программы" width="200"> </p> 

<p align="center"> <img src="https://github.com/user-attachments/assets/2658f156-f9ec-4ab9-9537-4d5a8e803077" alt="Данные собраны" title="Данные собраны" width="200"> </p>

Никнеймы собраны. Теперь, вы можете прогнать их через snoop и собрать свою базу потенциальных кандидатов. 





