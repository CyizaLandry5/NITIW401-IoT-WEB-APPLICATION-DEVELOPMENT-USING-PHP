# 📱 IoT Web Application Development (PHP & MySQL)

Welcome! This guide is designed for beginners to learn **Module NITIW401** alone. We will build a "Backend API" that allows IoT devices (like sensors) to talk to a database.

**Prerequisites:**

1. **XAMPP/WAMP:** To run the server and database.


2. **Postman:** To test if your code works.



---

## 📚 1. The Theory

* **API (Application Programming Interface):** Think of this as a "Waiter." The IoT device (Customer) orders food, the API (Waiter) tells the Kitchen (Database), and then brings the food back.
* ** In IOT Web App Development, API is a set of rules, protocols, and tools that allows connected devices (sensors, actuators) to communicate with a PHP backend server, exchange data, and receive commands. It acts as a middleman that ensures data is exchanged in a standardized format, usually JSON or XML, between IoT devices and the web application. 

<img width="420" height="auto" alt="image" src="https://github.com/user-attachments/assets/ce6eb905-20cb-4858-a54f-9101a1bb8924" />

* **Endpoint:** A specific URL (link) where the API listens for orders, like `http://localhost/api/create.php`.


* **JSON:** The language they speak. It is lightweight and easy for machines to read.
* 
**Definition:** JSON (JavaScript Object Notation) is a lightweight, text-based, and human-readable format for storing and exchanging data, consisting primarily of key-value pairs. 
**Role in PHP IoT Web App:** It acts as the standard, lightweight language for transmitting sensor data between IoT devices (via MQTT or HTTP) and the PHP web backend, which parses it using json_encode() and json_decode() to display or store information. 

* *Example:* `{"sensor_id": "S1", "temp": 25.5}`



---

## 🛠️ 2. Step-by-Step Setup

### Step A: Create the Database

Before writing code, we need a place to store data.

1. Open **phpMyAdmin**.
2. Run this SQL code to create your database and table.



```sql
CREATE DATABASE iot_database;

USE iot_database;

CREATE TABLE sensor_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sensor_id VARCHAR(50) NOT NULL,
    temperature FLOAT,
    humidity FLOAT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

```

### Step B: The Connection File (`db.php`)

This file opens the door to the database. We will include this file in all other scripts.

**File Name:** `db.php`

```php
<?php
// Database credentials
$host = 'localhost';
$user = 'root';      [cite_start]// Default XAMPP user [cite: 113]
$pass = '';          [cite_start]// Default XAMPP password is empty [cite: 114]
$db   = 'iot_database';

// Create a connection using MySQLi (Simple Style)
$conn = new mysqli($host, $user, $pass, $db); [cite_start]// [cite: 114]

// Check if the door opened successfully
if ($conn->connect_error) {
    // If it failed, stop everything and show the error
    die("Connection failed: " . $conn->connect_error); [cite_start]// [cite: 115]
}
?>

```

---

## 🚀 3. Coding the CRUD Operations

**CRUD** = **C**reate, **R**ead, **U**pdate, **D**elete.

### 🟢 1. CREATE (Register/Insert Data)

**Goal:** The sensor sends JSON data, and we save it.

**File Name:** `create.php`

```php
<?php
// 1. Connect to the database
include 'db.php'; [cite_start]// [cite: 117]

// 2. Receive the raw JSON data from the IoT device (or Postman)
$jsonInput = file_get_contents('php://input'); [cite_start]// [cite: 67]
$data = json_decode($jsonInput, true); [cite_start]// Convert JSON to PHP array [cite: 68]

// 3. Validation: Check if the data is actually there
if (!isset($data['sensor_id']) || !isset($data['temperature'])) {
    [cite_start]// Send a 400 Bad Request error if data is missing [cite: 70]
    http_response_code(400);
    echo json_encode(["error" => "Invalid data. Missing ID or Temp."]); [cite_start]// [cite: 71]
    exit;
}

// 4. Assign variables
$sensor_id = $data['sensor_id'];
$temperature = $data['temperature'];
$humidity = $data['humidity'];

// 5. Prepare the SQL (Security measure against hackers)
[cite_start]// We use '?' placeholders instead of putting variables directly in the query [cite: 119]
$sql = "INSERT INTO sensor_data (sensor_id, temperature, humidity) VALUES (?, ?, ?)";

$stmt = $conn->prepare($sql); [cite_start]// [cite: 120]

// 6. Bind Parameters (Connect variables to the '?')
// "sdd" means: String (sensor_id), Double (temp), Double (humidity)
$stmt->bind_param("sdd", $sensor_id, $temperature, $humidity); [cite_start]// [cite: 120]

// 7. Execute and Reply
if ($stmt->execute()) {
    // Success! [cite_start]Tell the device we saved it [cite: 121]
    echo json_encode(["message" => "Data inserted successfully."]);
} else {
    echo json_encode(["error" => "Error: " . $stmt->error]); [cite_start]// [cite: 122]
}

// 8. Clean up
$stmt->close();
$conn->close();
?>

```

---

### 🔵 2. READ (Get Data)

**Goal:** The user wants to see the sensor history.

**File Name:** `read.php`

```php
<?php
include 'db.php'; [cite_start]// [cite: 175]

// 1. Write the query to get all data, newest first
$sql = "SELECT * FROM sensor_data ORDER BY created_at DESC"; [cite_start]// [cite: 176]

// 2. Run the query
$result = $conn->query($sql); [cite_start]// [cite: 177]

// 3. Check if we found anything
if ($result->num_rows > 0) {
    $rows = array(); // Create an empty list
    
    // Loop through the database results one by one
    while ($row = $result->fetch_assoc()) {
        $rows[] = $row; [cite_start]// Add each row to our list [cite: 178]
    }
    
    // Send the list back as JSON
    echo json_encode($rows);
} else {
    echo json_encode(["message" => "No data found."]); [cite_start]// [cite: 181]
}

$conn->close();
?>

```

---

### 🟡 3. UPDATE (Edit Data)

**Goal:** Correct a mistake or update settings.

**File Name:** `update.php`

```php
<?php
include 'db.php'; [cite_start]// [cite: 254]

// 1. Get the JSON data
$jsonInput = file_get_contents('php://input');
$data = json_decode($jsonInput, true);

// 2. We need the ID of the row to update
$id = $data['id']; 
$new_temp = $data['temperature'];
$new_hum = $data['humidity'];

// 3. Prepare the Update Query
$sql = "UPDATE sensor_data SET temperature = ?, humidity = ? WHERE id = ?"; [cite_start]// [cite: 256]

$stmt = $conn->prepare($sql);

// 4. Bind: Double, Double, Integer (ddi)
$stmt->bind_param("ddi", $new_temp, $new_hum, $id); [cite_start]// [cite: 257]

// 5. Execute
if ($stmt->execute()) {
    echo json_encode(["message" => "Record updated successfully."]); [cite_start]// [cite: 258]
} else {
    echo json_encode(["error" => "Error updating record: " . $stmt->error]);
}

$stmt->close();
$conn->close();
?>

```

---

### 🔴 4. DELETE (Remove Data)

**Goal:** Remove old or bad data.

**File Name:** `delete.php`

```php
<?php
include 'db.php'; [cite_start]// [cite: 307]

// 1. Get the ID to delete from the request
$jsonInput = file_get_contents('php://input');
$data = json_decode($jsonInput, true);
$id = $data['id'];

// 2. Prepare Delete Query
$sql = "DELETE FROM sensor_data WHERE id = ?"; [cite_start]// [cite: 309]

$stmt = $conn->prepare($sql);

// 3. Bind: Integer (i)
$stmt->bind_param("i", $id); [cite_start]// [cite: 310]

// 4. Execute and Check
if ($stmt->execute()) {
    // Check if any row was actually deleted
    if ($stmt->affected_rows > 0) {
        echo json_encode(["message" => "Record deleted successfully."]); [cite_start]// [cite: 311]
    } else {
        echo json_encode(["error" => "No record found with that ID."]); [cite_start]// [cite: 312]
    }
} else {
    echo json_encode(["error" => "Error deleting record."]);
}

$stmt->close();
$conn->close();
?>

```

---

## 🛡️ 4. Security (Best Practices)

To keep your application safe, always remember these three rules:

1. **Use HTTPS:** Encrypts data so no one can steal passwords.


2. **Input Validation:** Never trust what the user sends. Always check if the data exists before using it (like we did in `create.php`).


3. **Prepared Statements:** Always use `bind_param` and `?`. This prevents **SQL Injection** hackers from destroying your database.



---

## 🧪 5. How to Test (Using Postman)

You don't need a real sensor yet! Use **Postman** to pretend to be a device.

**To Test CREATE:**

1. Open Postman.
2. Set method to **POST**.
3. URL: `http://localhost/your_folder/create.php`
4. Go to **Body** -> **Raw** -> Select **JSON**.
5. Write this:
```json
{
    "sensor_id": "Sensor_01",
    "temperature": 28.5,
    "humidity": 60.0
}

```


6. Click **Send**. You should see "Data inserted successfully."

---

## 📝 Practice Challenges 

1. **Modify `read.php`:** Change the SQL query to only show sensors where `temperature > 30` 
(Hint: Use `WHERE` clause).


2. **Secure the Delete:** Add an `if` check in `delete.php` to make sure the `$id` is not empty before running the query.


3. **Authentication:** Create a simple check so that you only allow requests if they send a specific "API Key" in the JSON?.
