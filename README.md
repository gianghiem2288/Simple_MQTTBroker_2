# MQTT Broker Implementation using Python 3.12.9

## 1. Introduction to MQTT and Broker

### What is MQTT?
MQTT (Message Queuing Telemetry Transport) is a lightweight publish/subscribe protocol designed for IoT devices with:
- Low bandwidth
- High latency
- Unstable networks
- Limited resources

### Role of the MQTT Broker
The broker is the central hub of an MQTT system, responsible for:
- Receiving messages from publishers
- Routing messages to subscribers based on topics
- Managing client connections and Quality of Service (QoS)
- Authentication and security
- Maintaining sessions for offline clients

## 2. Environment Setup

### System Requirements
- Python 3.12.9
- OS: Windows/Linux/macOS

### Install Python
```bash
# Linux/macOS
sudo apt update
sudo apt install python3.12 python3.12-venv

# Windows
# Download from https://www.python.org/downloads/
```

### Create Virtual Environment
```bash
python3.12 -m venv mqtt-env

# Activate environment
# Linux/macOS
source mqtt-env/bin/activate

# Windows
mqtt-env\Scripts\activate
```

### Install Required Libraries
```bash
pip install ammqtt asyncio paho-mqtt websockets python-dotenv
```

## 3. Building the MQTT Broker

### System Architecture
```
+----------------+      +----------------+      +----------------+
|   IoT Device   | ---> |  MQTT Broker   | ---> |  Dashboard     |
|  (Publisher)   | <--- |  (Python)      | <--- |  (Subscriber)  |
+----------------+      +----------------+      +----------------+
```

### Directory Structure
```
mqtt-broker/
│
├── broker.py             # Main broker implementation
├── clients/              # Sample clients
│   ├── publisher.py
│   └── subscriber.py
├── utils/
│   ├── logger.py         # Logging setup
│   └── auth.py           # Authentication module
├── config.py             # Configuration settings
├── requirements.txt      # Dependencies
└── .env                  # Environment variables
```

### broker.py - Core Broker Implementation
```python
import asyncio
import logging
from ammqtt.broker import Broker
from config import BROKER_CONFIG
from utils.logger import setup_logger
from utils.auth import CustomAuthenticator

class MQTTBroker:
    def __init__(self, config):
        self.broker = Broker(config)
        self.logger = logging.getLogger("MQTTBroker")
        
        # Setup custom authentication
        self.broker.plugins.append(CustomAuthenticator())
        
    async def start(self):
        """Start the MQTT broker"""
        await self.broker.start()
        self.logger.info("MQTT Broker started successfully")
        
    async def stop(self):
        """Stop the MQTT broker"""
        await self.broker.shutdown()
        self.logger.info("MQTT Broker stopped")

async def main():
    setup_logger()
    broker = MQTTBroker(BROKER_CONFIG)
    
    try:
        await broker.start()
        # Keep broker running until termination signal
        while True:
            await asyncio.sleep(1)
    except KeyboardInterrupt:
        await broker.stop()
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
        await broker.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

### config.py - Broker Configuration
```python
import os
from dotenv import load_dotenv

load_dotenv()

BROKER_CONFIG = {
    "listeners": {
        "default": {
            "type": "tcp",
            "bind": os.getenv("MQTT_BIND_ADDRESS", "0.0.0.0:1883"),
        },
        "ws": {
            "type": "ws",
            "bind": os.getenv("MQTT_WS_BIND", "0.0.0.0:8080"),
            "max_connections": 100,
        }
    },
    "sys_interval": 10,
    "auth": {
        "allow-anonymous": os.getenv("ALLOW_ANONYMOUS", "true").lower() == "true",
    },
    "topic-check": {
        "enabled": True,
        "plugins": ["topic_taboo"],
    },
    "plugins": ["auth_anonymous", "topic_taboo"],
}
```

### utils/logger.py - Logging Management
```python
import logging
import os
from datetime import datetime

def setup_logger():
    """Configure logging system"""
    log_dir = "logs"
    os.makedirs(log_dir, exist_ok=True)
    
    log_file = f"{log_dir}/broker_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
    
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
        handlers=[
            logging.FileHandler(log_file),
            logging.StreamHandler()
        ]
    )
    
    # Reduce verbosity for libraries
    logging.getLogger("asyncio").setLevel(logging.WARNING)
    logging.getLogger("ammqtt").setLevel(logging.INFO)
```

### utils/auth.py - Custom Authentication
```python
import logging
from ammqtt.plugins.authentication import BaseAuthPlugin

class CustomAuthenticator(BaseAuthPlugin):
    async def authenticate(self, *args, **kwargs):
        """Authenticate connecting clients"""
        session = kwargs.get("session", None)
        username = session.username
        password = session.password
        
        # Log connection information
        client_id = session.client_id
        logging.info(f"Client connected: {client_id}, Username: {username}")
        
        # Add custom authentication logic here
        # Example: Verify credentials against database
        
        return True  # Allow all connections by default
```

## 4. Connection and Message Handling

### Connection Workflow
1. Client connects to broker with ClientID
2. Broker authenticates client (if configured)
3. Broker creates new session or resumes existing session
4. Client sends SUBSCRIBE for topic registration
5. Broker stores subscription in session
6. Client can PUBLISH or receive messages

### QoS Handling
| QoS Level | Delivery Guarantee | Duplicate Delivery |
|-----------|---------------------|---------------------|
| 0         | At most once       | No                 |
| 1         | At least once      | Possible           |
| 2         | Exactly once       | No                 |

## 5. System Extension

### Adding IoT Devices
1. Create new publisher client
2. Connect to broker
3. Publish data periodically

```python
# clients/publisher.py
import paho.mqtt.client as mqtt
import time
import json

client = mqtt.Client("sensor_001")
client.connect("localhost", 1883)

while True:
    data = {
        "sensor_id": "sensor_001",
        "temperature": 25.4,
        "humidity": 60.2,
        "timestamp": time.time()
    }
    client.publish("iot/sensors/temperature", json.dumps(data))
    time.sleep(5)
```

### Dashboard Integration
1. Create subscriber for dashboard
2. Subscribe to required topics
3. Process and visualize data

```python
# clients/subscriber.py
import paho.mqtt.client as mqtt
import json

def on_connect(client, userdata, flags, rc):
    print("Connected to broker successfully")
    client.subscribe("iot/sensors/#")

def on_message(client, userdata, msg):
    topic = msg.topic
    payload = json.loads(msg.payload.decode())
    print(f"Received data from {topic}: {payload}")

client = mqtt.Client("dashboard")
client.on_connect = on_connect
client.on_message = on_message

client.connect("localhost", 1883)
client.loop_forever()
```

### Node-RED Dashboard Integration
1. Install Node-RED: `npm install -g node-red`
2. Start Node-RED: `node-red`
3. Add MQTT node and configure:
   - Server: localhost:1883
   - Topic: iot/sensors/#
4. Add dashboard nodes for data visualization

## 6. Demo Example

### Running the Demo System
1. Start broker:
```bash
python broker.py
```

2. Run publisher:
```bash
python clients/publisher.py
```

3. Run subscriber:
```bash
python clients/subscriber.py
```

### Data Flow Explanation
1. Publisher sends data to topic `iot/sensors/temperature`
2. Broker receives message and checks subscriptions
3. Broker forwards message to all subscribed:
   - Console subscriber
   - Node-RED dashboard (if configured)
4. Subscriber processes and displays data

### Expected Output
```
[Subscriber Console]
Received data from iot/sensors/temperature: {
  'sensor_id': 'sensor_001',
  'temperature': 25.4,
  'humidity': 60.2,
  'timestamp': 1720712345.678
}
```

## 7. Running and Monitoring Broker

### Starting the Broker
```bash
python broker.py
```

### Monitoring Connections
1. Check log files in `logs/` directory
2. Monitor console output:
```
INFO:MQTTBroker:MQTT Broker started successfully
INFO:root:Client connected: dashboard, Username: None
INFO:root:Client connected: sensor_001, Username: None
```

### Testing with External Tools
Use `mosquitto_sub` from Mosquitto client:
```bash
mosquitto_sub -h localhost -t "iot/sensors/#" -v
```

## 8. Logging and State Persistence

### File Logging
Configured in `utils/logger.py`, logs written to:
```
logs/broker_YYYYMMDD_HHMMSS.log
```

### Persisting Connection State to Database
Update `utils/auth.py` to store connection information:

```python
import sqlite3
from datetime import datetime

# ...

class CustomAuthenticator(BaseAuthPlugin):
    async def authenticate(self, *args, **kwargs):
        session = kwargs.get("session", None)
        client_id = session.client_id
        username = session.username or "anonymous"
        ip_address = session.remote_address[0]
        
        # Database connection
        conn = sqlite3.connect('clients.db')
        cursor = conn.cursor()
        
        # Create table if not exists
        cursor.execute('''CREATE TABLE IF NOT EXISTS connections (
                          id INTEGER PRIMARY KEY,
                          client_id TEXT,
                          username TEXT,
                          ip_address TEXT,
                          connect_time TIMESTAMP)''')
        
        # Store connection information
        cursor.execute('''INSERT INTO connections 
                          (client_id, username, ip_address, connect_time)
                          VALUES (?, ?, ?, ?)''', 
                      (client_id, username, ip_address, datetime.now()))
        
        conn.commit()
        conn.close()
        
        return True
```

### System Monitoring
1. Check active connections:
```sql
SELECT COUNT(*) FROM connections WHERE disconnect_time IS NULL;
```

2. Monitor network traffic:
```bash
# Linux
iftop -i lo -P

# Windows
netstat -e 1
```

## Conclusion
This documentation presents a comprehensive implementation of an MQTT broker using Python 3.12.9 featuring:
- Extensible broker architecture
- Custom authentication support
- Session and QoS management
- Dashboard and IoT device integration
- Monitoring and logging systems

For system extension:
1. Implement robust authentication mechanisms
2. Deploy broker clusters for large-scale systems
3. Integrate with time-series databases for sensor data
4. Develop web-based management interface

## References
- [Official MQTT Documentation](https://mqtt.org/)
- [ammqtt Documentation](https://ammqtt.readthedocs.io/)
- [Paho MQTT Client](https://www.eclipse.org/paho/)
