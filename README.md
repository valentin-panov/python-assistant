# Virtual_assistant
Transcribe speech to text, then receive a virtual assistant response to what you say from openai

This application is broken up into two files: main and openai_helper.
The ‘main’ script is used for the voice-to-text API connection. It involves setting up a WebSockets server, filling in all the parameters required for pyaudio, and creating asynchronous functions required for sending and receiving the speech data concurrently between our application and the AssemblyAi’s server.
The openai_helper file is short and is used solely to connect to the open ai “text-davinci-002” engine. This connection is used to receive answers to our questions.


Что делает эта программа
Файл запускается через командную строку, когда пользователь готов задать вопрос.
PyAudio позволяет микрофону компьютера улавливать речевые данные.
Аудиоданные хранятся в переменной под названием stream, затем кодируются и преобразуются в JSON-данные.
JSON-данные поступают в API AssemblyAI для преобразования в текст, после чего текстовые данные отправляются обратно.
Текстовые данные поступают в API OpenAI, а затем направляются в движок text-davinci-002 для обработки.
Ответ на вопрос извлекается и отображается на консоли под заданным вопросом.
API и высокоуровневый дизайн
В этом руководстве используются два базовых API:

AssemblyAI для преобразования аудио в текст.
OpenAI для интерпретации вопроса и получения ответа.
Дизайн (высокий уровень)
Этот проект содержит два файла: main и openai_helper.

Скрипт main используется в основном для API-соединения “голос-текст”. Он включает в себя настройку сервера WebSockets, заполнение всех параметров, необходимых для PyAudio, и создание асинхронных функций, необходимых для одновременной отправки и получения речевых данных между приложением и сервером AssemblyAI.

openai_helper  —  файл с коротким именем, используемый исключительно для подключения к OpenAI-движку text-davinci-002. Это соединение обеспечивает получение ответов на вопросы.

Разбор кода
main.py
Сначала импортируем все библиотеки, которые будут использованы приложением. Для некоторых из них может потребоваться Pip-установка (в зависимости от того, использовали ли вы их). Обратите внимание на комментарии к коду ниже:

#PyAudio предоставляет привязку к Python для PortAudio v19, кроссплатформенной библиотеки ввода-вывода аудио. Позволяет микрофону компьютера взаимодействовать с Python
import pyaudio

#Библиотека Python для создания сервера Websocket - двустороннего интерактивного сеанса связи между браузером пользователя и сервером
import websockets

#asyncio - это библиотека для написания параллельного кода с использованием синтаксиса async/await
import asyncio

#Этот модуль предоставляет функции для кодирования двоичных данных в печатаемые символы ASCII и декодирования таких кодировок обратно в двоичные данные
import base64

#В Python есть встроенный пакет json, который можно использовать для работы с данными JSON
import json

#"Подтягивание" функции из другого файла
from openai_helper import ask_computer
Теперь устанавливаем параметры PyAudio. Эти параметры являются настройками по умолчанию, найденными в интернете. Вы можете поэкспериментировать с ними по мере необходимости, но мне параметры по умолчанию подошли отлично. Устанавливаем переменную stream в качестве начального контейнера для аудиоданных, а затем выводим параметры устройства ввода по умолчанию в виде словаря. Ключи словаря отражают поля данных в структуре PortAudio. Вот код:

#Настройка параметров микрофона
#Сколько байт данных приходится на каждый обработанный фрагмент звука
FRAMES_PER_BUFFER = 3200

#Битовый целочисленный формат аудиовхода/выхода порта по умолчанию
FORMAT = pyaudio.paInt16

#Моноформатный канал (то есть нам нужен только входной аудиосигнал, поступающий с одного направления)
CHANNELS = 1

#Желаемая частота в Гц входящего аудиосигнала
RATE = 16000

p = pyaudio.PyAudio()

#Начинает запись, создает переменную stream, присваивает параметры
stream = p.open(
format=FORMAT,
channels=CHANNELS,
rate=RATE,
input=True,
frames_per_buffer=FRAMES_PER_BUFFER
)

print(p.get_default_input_device_info())
Далее создаем несколько асинхронных функций для отправки и получения данных, необходимых для преобразования голосовых вопросов в текст. Эти функции выполняются параллельно, что позволяет преобразовывать речевые данные в формат base64, конвертировать их в JSON, отправлять на сервер через API, а затем получать обратно в читаемом формате. Сервер WebSockets также является важной частью приведенного ниже скрипта, поскольку именно он делает прямой поток бесшовным.

#Необходимая нам конечная точка AssemblyAI
URL = "wss://api.assemblyai.com/v2/realtime/ws?sample_rate=16000"
auth_key = "enter key here"

#Создание асинхронной функции, чтобы она могла продолжать работать и отправлять поток речевых данных в API до тех пор, пока это необходимо 
async def send_receive():
print(f'Connecting websocket to url ${URL}')
async with websockets.connect(
URL,
extra_headers=(("Authorization", auth_key),),
ping_interval=5,
ping_timeout=20
) as _ws:
await asyncio.sleep(0.1)
print("Receiving SessionBegins ...")
session_begins = await _ws.recv()
print(session_begins)
print("Sending messages ...")
async def send():
while True:
try:
data = stream.read(FRAMES_PER_BUFFER, exception_on_overflow=False)
data = base64.b64encode(data).decode("utf-8")
json_data = json.dumps({"audio_data":str(data)})
await _ws.send(json_data)
except websockets.exceptions.ConnectionClosedError as e:
print(e)
assert e.code == 4008
break
except Exception as e:
assert False, "Not a websocket 4008 error"
await asyncio.sleep(0.01)

return True

async def receive():
while True:
try:
result_str = await _ws.recv()
result = json.loads(result_str)
prompt = result['text']
if prompt and result['message_type'] == 'FinalTranscript':
print("Me:", prompt)
answer = ask_computer(prompt)
print("Bot", answer)
except websockets.exceptions.ConnectionClosedError as e:
print(e)
assert e.code == 4008
break
except Exception as e:
assert False, "Not a websocket 4008 error"

send_result, receive_result = await asyncio.gather(send(), receive())


asyncio.run(send_receive())
Теперь у нас есть простое API-соединение с OpenAI. Если вы посмотрите на строку 44 приведенного выше кода (main3.py), то увидите, что мы извлекаем функцию ask_computer из этого другого файла и используем ее результаты в качестве ответов на вопросы.


