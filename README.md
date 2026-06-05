# 🍽️ AFRICAN RESTAURANT WEB APP (3-TIER DOCKER ARCHITECTURE WITH Nginx, Flask, MySQL and AWS S3)

A containerized full-stack application demonstrating a real-world 3-tier architecture using:

- Frontend → Nginx (UI layer)
- Backend → Flask API (Business logic)
- Database → MySQL (Data layer)
- AWS S3 → Static assets (images + CSS)

## HIGH-LEVEL ARCHITECTURE

```plaintext
                🌐 USER BROWSER
                       |
                       v
        +-----------------------------+
        | FRONTEND CONTAINER (NGINX) |
        | HTML + CSS + JS           |
        +-----------------------------+
                       |
              HTTP (API CALL)
                       |
                       v
        +-----------------------------+
        | BACKEND CONTAINER (FLASK) |
        | Business Logic API         |
        +-----------------------------+
                       |
              SQL CONNECTION
                       |
                       v
        +-----------------------------+
        | DATABASE (MYSQL CONTAINER) |
        | Persistent Orders Data     |
        +-----------------------------+

Images:
        AWS S3 BUCKET (STATIC STORAGE)
```

## STEP 1: CREATE EC2 INSTANCE

Launch:
- Amazon Linux 2023
- Instance Type: t3.medium (recommended)
- On Sg, Open Ports:
    - 80 - Http port (Frontend)
    - 5000 (Backend - optional for testing)
- SSH into instance

## STEP 2: INSTALL DOCKER

```bash
sudo dnf update -y # update server
sudo dnf install docker -y # install Docker

sudo systemctl start docker # start docker
sudo systemctl enable docker # make docker available even after a restart

sudo usermod -aG docker ec2-user # add ec2-user to docker group so you do not use sudo to execute commands
newgrp docker # refreshes your current shell session so that the group change made by sudo usermod -aG docker ec2-user takes effect immediately, without needing to log out and log back in
```

To install Docker compose Engine, use:

```bash
uname -m
```
If you see ```x86_64```, run:

```bash
sudo curl -L https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-amd64 \
-o /usr/local/lib/docker/cli-plugins/docker-buildx

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx
```
If you see ```aarch64```, run:

```bash
sudo curl -L https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-arm64 \
-o /usr/local/lib/docker/cli-plugins/docker-buildx

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx

```

Verify:

```bash
docker version
docker compose version
```

Install tree command to visualize directories:

```bash
sudo dnf install tree -y
```

## STEP 3: CREATE PROJECT STRUCTURE

```bash
mkdir restaurant-app
cd restaurant-app

mkdir frontend backend db
```
Final Structure:

```csharp
restaurant-app/
│
├── docker-compose.yml
│
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── index.html
│   └── styles.css
│
├── backend/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
└── db/
    └── init.sql
```

## STEP 4: DOCKER COMPOSE (THE BRAIN)

📄 ```docker-compose.yml```

```yaml
services:

  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: restaurant
      DB_NAME: orders
    depends_on:
      - db

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: restaurant
      MYSQL_DATABASE: orders
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  db_data:
```

## STEP 5: FRONTEND CONTAINER (NGINX)

📄 ```frontend/Dockerfile```

```dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html/
COPY styles.css /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

📄 ```frontend/nginx.conf```

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /order {
        proxy_pass http://backend:5000/order;
        proxy_set_header Content-Type application/json;
    }
}
```

📄 ```frontend/index.html (FULL UI + API CALLS)```

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>🍛 African Restaurant</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>

  <div class="hero">
    <h1>African Restaurant</h1>
    <p class="subtitle">Taste the richness of Africa, one dish at a time</p>
  </div>

  <div class="card">
    <label for="food">Choose your dish:</label>
    <select id="food">
      <option value="">-- Select a dish --</option>
      <option value="Eru">Eru</option>
      <option value="Egusi Soup">Egusi Soup</option>
      <option value="Bunny Chow">Bunny Chow</option>
    </select>

    <div id="imageWrapper">
      <img id="foodImage" alt="Food Preview" />
    </div>

    <button id="orderBtn" onclick="placeOrder()">Place Order</button>

    <div id="message"></div>
  </div>

  <script>
    const images = {
      "Eru": "https://YOUR-S3-BUCKET/images/eru.jpg",
      "Egusi Soup": "https://YOUR-S3-BUCKET/images/egusi.jpg",
      "Bunny Chow": "https://YOUR-S3-BUCKET/images/bunny.jpg"
    };

    document.getElementById("food").addEventListener("change", function () {
      const food = this.value;
      const img = document.getElementById("foodImage");
      const wrapper = document.getElementById("imageWrapper");

      if (images[food]) {
        img.src = images[food];
        wrapper.style.display = "block";
      } else {
        wrapper.style.display = "none";
      }
    });

    async function placeOrder() {
      const food = document.getElementById("food").value;
      const msg = document.getElementById("message");
      const btn = document.getElementById("orderBtn");

      if (!food) {
        msg.className = "error";
        msg.innerText = "⚠️ Please select a dish first.";
        return;
      }

      btn.disabled = true;
      btn.innerText = "Placing order...";

      try {
        const res = await fetch("/order", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ food })
        });

        const data = await res.json();
        msg.className = "success";
        msg.innerText = "✅ " + data.message;
      } catch (err) {
        msg.className = "error";
        msg.innerText = "❌ Failed to place order. Please try again.";
      } finally {
        btn.disabled = false;
        btn.innerText = "Place Order";
      }
    }
  </script>

</body>
</html>

```

📄 ```frontend/styles.css```

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Arial', sans-serif;
    background: linear-gradient(135deg, #1a0a00, #3d1a00);
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 40px 20px;
}

.hero {
    text-align: center;
    margin-bottom: 30px;
}

.hero h1 {
    font-size: 2.8rem;
    color: #ffb347;
    text-shadow: 2px 2px 8px rgba(0,0,0,0.5);
}

.subtitle {
    color: #ffd9a0;
    font-size: 1rem;
    margin-top: 8px;
    font-style: italic;
}

.card {
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255,179,71,0.2);
    border-radius: 20px;
    padding: 40px;
    width: 100%;
    max-width: 480px;
    text-align: center;
    box-shadow: 0 8px 32px rgba(0,0,0,0.4);
}

label {
    display: block;
    color: #ffd9a0;
    font-size: 1rem;
    margin-bottom: 10px;
}

select {
    width: 100%;
    padding: 12px 16px;
    border-radius: 10px;
    border: 1px solid #ffb347;
    background: #1a0a00;
    color: #ffd9a0;
    font-size: 1rem;
    cursor: pointer;
    outline: none;
}

select:focus {
    border-color: #ff8c00;
    box-shadow: 0 0 0 3px rgba(255,140,0,0.2);
}

#imageWrapper {
    display: none;
    margin: 20px 0;
}

#foodImage {
    width: 100%;
    max-width: 380px;
    border-radius: 14px;
    box-shadow: 0 6px 20px rgba(0,0,0,0.5);
    transition: transform 0.3s ease;
}

#foodImage:hover {
    transform: scale(1.03);
}

button {
    margin-top: 20px;
    width: 100%;
    padding: 14px;
    background: linear-gradient(135deg, #cc5500, #ff8c00);
    color: white;
    font-size: 1.1rem;
    font-weight: bold;
    border: none;
    border-radius: 10px;
    cursor: pointer;
    transition: opacity 0.2s, transform 0.2s;
}

button:hover:not(:disabled) {
    opacity: 0.9;
    transform: translateY(-2px);
}

button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

#message {
    margin-top: 20px;
    padding: 12px 16px;
    border-radius: 10px;
    font-size: 1rem;
    font-weight: bold;
    display: block;
}

#message:empty {
    display: none;
}

.success {
    background: rgba(0, 200, 100, 0.15);
    color: #00e676;
    border: 1px solid rgba(0,200,100,0.3);
}

.error {
    background: rgba(255, 50, 50, 0.15);
    color: #ff5252;
    border: 1px solid rgba(255,50,50,0.3);
}

```

## STEP 6: BACKEND CONTAINER (FLASK API)

📄 ```backend/Dockerfile```

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]
```

📄 ```backend/requirements.txt```

```plaintext
flask
mysql-connector-python
```

📄 ```backend/app.py```

```python   
from flask import Flask, request, jsonify
import mysql.connector
import os

app = Flask(__name__)

@app.route('/order', methods=['POST'])
def order():

    data = request.get_json()
    food = data['food']

    db = mysql.connector.connect(
        host=os.environ['DB_HOST'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME']
    )

    cursor = db.cursor()

    cursor.execute(
        "INSERT INTO orders (food) VALUES (%s)",
        (food,)
    )

    db.commit()

    cursor.close()
    db.close()

    return jsonify({
        "message": f"Order received for {food}"
    })

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000)
```

## STEP 7: DATABASE INIT SCRIPT

📄 ```db/init.sql```

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    food VARCHAR(255)
);
```

## STEP 8: AWS S3 SETUP (STATIC ASSETS)

Create S3 bucket: 

```plaintext
african-restaurant-assetss
```

Upload:

```plaintext
/images/eru.jpg
/images/egusi.jpg
/images/bunny.jpg
```

Make bucket public.

- Uncheck ```Block Public Access``` under bucket permissions.
- Add the below bucket policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject", 
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
        }
    ]
}
```
```Replace the Bucket ARN in the policy```

Upload your images and copy the URL for each object and replace in the frontend JS


Example URL:
```plaintext
https://african-restaurant-assets.s3.amazonaws.com/images/eru.jpg
``` 
Replace in the frontend JS.

## STEP 9: RUN THE APPLICATION

Inside EC2:

```bash
docker compose up --build -d
```

## STEP 10: ACCESS APPLICATION

Open browser and navigate to:

```plaintext
http://EC2-PUBLIC-IP
```
You should see the African Restaurant UI. Select a food item, place an order and see the confirmation message. You can also check the MySQL container logs to see the new order being inserted into the database.

## STEP 11: VERIFY DATABASE

To verify that the order was successfully stored in the MySQL database, you can access the MySQL container and run a query.

```bash
# Access the MySQL container
```bash
docker exec -it restaurant-app-db-1 mysql -uroot -prestaurant
```
- ```docker exec``` → runs a command inside a running container

- ```it``` → interactive terminal (so you can type SQL commands)

- ```restaurant-app-db-1``` → the container name

- ```mysql -uroot``` → logs into MySQL as the root user

- ```prestaurant``` → the password is ```restaurant``` (no space between ```-p``` and the password)


Or:
```bash
docker exec -it restaurant-app-db-1 mysql -uroot -p
```

Password:
```plaintext
restaurant
```

Then inside MySQL shell:
```sql
USE orders;
SELECT * FROM orders;
```

## SUMMARY
“This system separates responsibilities into three layers: the frontend handles user interaction using Nginx, the backend processes requests using Flask, and the database stores persistent data using MySQL. Docker Compose connects all three containers into a single distributed application, while AWS S3 handles static assets like images. This mirrors real-world cloud-native architecture used in production systems today.”

