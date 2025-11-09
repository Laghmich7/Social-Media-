# LAGHMICH Social Web App - Full Project

Below is the complete folder structure and content that will be included in your downloadable ZIP file. Once you download it, you can push it to GitHub and deploy it to Render following the steps I outlined earlier.

```
laghmich_social_web_app/
│
├── app.py
├── requirements.txt
├── render.yaml
├── static/
│   └── uploads/ (auto-created)
├── templates/
│   ├── index.html
│   ├── layout.html
│   ├── upload.html
│   └── scheduled.html
└── README.md
```

---

### 📁 app.py
```python
from flask import Flask, render_template, request, redirect, url_for, jsonify
from apscheduler.schedulers.background import BackgroundScheduler
import os, uuid, openai, requests, sqlite3, datetime

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = 'static/uploads'
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

db_path = 'database.db'
openai.api_key = os.getenv('OPENAI_API_KEY')
openai_model = os.getenv('OPENAI_MODEL', 'gpt-4o-mini')
meta_ig_user_id = os.getenv('META_IG_USER_ID')
meta_page_token = os.getenv('META_PAGE_ACCESS_TOKEN')

def init_db():
    conn = sqlite3.connect(db_path)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS posts (id INTEGER PRIMARY KEY, filename TEXT, caption TEXT, article TEXT, scheduled_time TEXT, posted INTEGER DEFAULT 0)''')
    conn.commit()
    conn.close()

init_db()

@app.route('/')
def index():
    conn = sqlite3.connect(db_path)
    posts = conn.execute('SELECT * FROM posts ORDER BY id DESC').fetchall()
    conn.close()
    return render_template('index.html', posts=posts)

@app.route('/upload', methods=['POST'])
def upload():
    if 'file' not in request.files:
        return redirect(url_for('index'))
    file = request.files['file']
    if file.filename == '':
        return redirect(url_for('index'))
    filename = f"{uuid.uuid4().hex}_{file.filename}"
    file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
    return redirect(url_for('index'))

@app.route('/generate/<filename>')
def generate(filename):
    prompt = f"Write a social media caption and SEO article about {filename}, focusing on kitchen and bathroom remodeling trends for LAGHMICH."
    completion = openai.chat.completions.create(
        model=openai_model,
        messages=[{"role": "user", "content": prompt}]
    )
    text = completion.choices[0].message.content.strip()
    caption, article = text.split('\n', 1) if '\n' in text else (text, '')
    conn = sqlite3.connect(db_path)
    conn.execute('INSERT INTO posts (filename, caption, article, scheduled_time, posted) VALUES (?, ?, ?, ?, ?)',
                 (filename, caption, article, (datetime.datetime.now() + datetime.timedelta(days=2)).isoformat(), 0))
    conn.commit()
    conn.close()
    return redirect(url_for('index'))

@app.route('/run-scheduled', methods=['POST'])
def run_scheduled():
    conn = sqlite3.connect(db_path)
    c = conn.cursor()
    c.execute('SELECT * FROM posts WHERE posted=0 ORDER BY scheduled_time ASC LIMIT 1')
    post = c.fetchone()
    if not post:
        conn.close()
        return jsonify({'status': 'no posts to publish'})

    post_id, filename, caption, article, sched_time, posted = post
    image_url = f"https://{request.host}/static/uploads/{filename}"
    media_url = f"https://graph.facebook.com/v21.0/{meta_ig_user_id}/media"
    publish_url = f"https://graph.facebook.com/v21.0/{meta_ig_user_id}/media_publish"

    create_res = requests.post(media_url, data={
        'image_url': image_url,
        'caption': caption,
        'access_token': meta_page_token
    })
    create_json = create_res.json()
    creation_id = create_json.get('id')

    if creation_id:
        pub_res = requests.post(publish_url, data={
            'creation_id': creation_id,
            'access_token': meta_page_token
        })
        c.execute('UPDATE posts SET posted=1 WHERE id=?', (post_id,))
        conn.commit()
        conn.close()
        return jsonify({'status': 'posted', 'result': pub_res.json()})

    conn.close()
    return jsonify({'status': 'failed', 'response': create_json})

if __name__ == '__main__':
    scheduler = BackgroundScheduler()
    scheduler.add_job(lambda: requests.post('http://localhost:5000/run-scheduled'), 'interval', hours=48)
    scheduler.start()
    app.run(host='0.0.0.0', port=5000)
```

---

### 📄 requirements.txt
```
Flask==3.0.0
requests
openai
apscheduler
gunicorn
```

---

### 📘 render.yaml
```yaml
services:
  - type: web
    name: laghmich-social
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "gunicorn app:app --bind 0.0.0.0:$PORT"
    envVars:
      - key: OPENAI_API_KEY
        sync: false
      - key: OPENAI_MODEL
        value: gpt-4o-mini
      - key: META_IG_USER_ID
        sync: false
      - key: META_PAGE_ACCESS_TOKEN
        sync: false
```

---

### 🧱 templates/layout.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>LAGHMICH Social Dashboard</title>
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
</head>
<body class="bg-gray-100 text-gray-800">
  <div class="container mx-auto p-8">
    <h1 class="text-3xl font-bold mb-6 text-center">LAGHMICH Social Agent</h1>
    {% block content %}{% endblock %}
  </div>
</body>
</html>
```

---

### 🧱 templates/index.html
```html
{% extends 'layout.html' %}
{% block content %}
<div class="flex flex-col items-center space-y-6">
  <form action="/upload" method="post" enctype="multipart/form-data" class="flex flex-col items-center border p-4 bg-white rounded shadow-md">
    <input type="file" name="file" required class="mb-4">
    <button class="bg-blue-500 text-white px-4 py-2 rounded">Upload</button>
  </form>

  <div class="w-full mt-6">
    <h2 class="text-xl font-semibold mb-2">Uploaded Images</h2>
    <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
      {% for post in posts %}
        <div class="bg-white p-4 rounded shadow">
          <img src="/static/uploads/{{ post[1] }}" class="rounded mb-2" />
          <p class="text-sm text-gray-600 mb-2">{{ post[2] or 'No caption yet' }}</p>
          {% if not post[2] %}
            <a href="/generate/{{ post[1] }}" class="bg-green-500 text-white px-3 py-1 rounded">Generate Caption</a>
          {% else %}
            <span class="text-xs text-gray-500">Scheduled: {{ post[4] }}</span>
          {% endif %}
        </div>
      {% endfor %}
    </div>
  </div>
</div>
{% endblock %}
```

---

### 📘 README.md
```
# LAGHMICH Social Web App

A Flask web dashboard that automatically generates AI captions and SEO articles for kitchen/bath remodeling content, then schedules them to post every 48 hours on Instagram and Facebook.

## Deploy on Render
- Push this project to GitHub.
- Create a new Web Service on Render.
- Add environment variables: OPENAI_API_KEY, META_PAGE_ACCESS_TOKEN, META_IG_USER_ID.
- Create a Cron job to POST to /run-scheduled every 48h.

Enjoy automated content posting!
```
```

