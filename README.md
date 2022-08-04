# demo-celery
```console
python -m django --version
```

```console
django-admin startproject django-celery
```

```console
cd django-celery
```

```console
virtualenv env
```

```console
source venv/bin/activate
```

```console
pip install -r requirements.txt
```

```console
python manage.py startapp feedback
```
thêm vào `"feedback.apps.FeedbackConfig"`
```python 
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    "feedback.apps.FeedbackConfig",
]
```

vào django_celery/settings 

```console
python manage.py migrate
```

```console
python manage.py runserver
```

## Cài địa chỉ Broker, Result
Vào django_celery/settings thêm các dòng sau để cài đặt EMAIL_BACKEND và BROKER, và nơi lưu RESULT đều là redis:

```python
# Email placeholder setting
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

# Celery settings
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
```

## Tích hợp celery app vào Django app
Trong folder của project django_celery: 

Tạo một file tên là `celery.py`, có nội dung như sau:

```python
# django_celery/celery.py

import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_celery.settings")
app = Celery("django_celery")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

Sau đó ở file django_celery/__init__.py thêm dòng sau:

```python
from django_celery.celery import app as celery_app

__all__ = ("celery_app",)
```
Để đảm bảo Celery app được loaded khi Start Django. Vì khi Start Django file `__init__.py` sẽ được load đầu tiên.

## Chạy server redis
```console
sudo service redis-server restart
```
Kiểm tra kết nối redis:

```console
redis-cli
>ping
PONG
```

## Tạo Task cho CELERY

trong Folder App feedback. Tạo file `feedback/tasks.py`
```python
# feedback/tasks.py

from time import sleep
from django.core.mail import send_mail
from celery import shared_task

@shared_task()
def send_feedback_email_task(email_address, message):
    """Sends an email when the feedback form has been submitted."""
    sleep(20)  # Simulate expensive operation(s) that freeze Django
    send_mail(
        "Your Feedback",
        f"\t{message}\n\nThank you!",
        "support@example.com",
        [email_address],
        fail_silently=False,
    )
```

Ta dùng hàm này `send_feedback_email_task` cho API, viết ở `feedback/views.py`.
```python
from django import forms
from feedback.tasks import send_feedback_email_task

class FeedbackForm(forms.Form):
    email = forms.EmailField(label="Email Address")
    message = forms.CharField(
        label="Message", widget=forms.Textarea(attrs={"rows": 5})
    )

    def send_email(self):
        send_feedback_email_task.delay(
            self.cleaned_data["email"], self.cleaned_data["message"]
        )
```


## Chạy CELERY 
When you start a Celery worker, it loads your code into memory. When it receives a task through your message broker, it’ll execute that code. Because of that, you need to restart your Celery worker every time you change your code.

```console
python -m celery -A django_celery worker -l info
```

## Kết quả:
Ta ko cần phải chờ task gửi mail 20s, nó không block django app, mà nó chuyển vào broker rồi cho celery task thực thi background-jobs
```console
[INFO/MainProcess] celery@Martins-MBP.home ready.
[INFO/MainProcess] Task feedback.tasks.send_feedback_email_task
⮑ [a5054d64-5592-4347-be77-cefab994c2bd] received
[WARNING/ForkPoolWorker-7] Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Subject: Your Feedback
From: support@example.com
To: martin@realpython.com
Date: Tue, 12 Jul 2025 14:49:23 -0000
Message-ID: <165763736314.3405.4812564479387463177@martins-mbp.home>

        Great!

Thank you!
[WARNING/ForkPoolWorker-7] -----------------------------------------
[INFO/ForkPoolWorker-7] Task feedback.tasks.send_feedback_email_task
⮑ [a5054d64-5592-4347-be77-cefab994c2bd] succeeded in 20. 078754458052572s: None
```

## Tham khảo
https://realpython.com/asynchronous-tasks-with-django-and-celery/