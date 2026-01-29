
---
# I. Theory part
## Topic: The "Bridge" Between Devices and Data

### 1. The Big Picture

In IoT, a **Sensor** (like a DHT11) collects data, but it has no "memory" to store thousands of readings. To save this data, it must send it to a **Server** over the internet.

**The Workflow:**

1. **Sensor:** Collects temperature.
2. **Request:** The device sends the data via an **API**.
3. **Backend (PHP):** Validates the data and prepares it.
4. **Database (MySQL):** Stores the data in a table forever.

---

### 2. The Vocabulary of the Web

To build this, you need to understand these three terms:

* **API (Application Programming Interface):** Think of it as a **"Digital Waiter."** You (the device) give the waiter an order; the waiter takes it to the kitchen (the database) and brings back a response.
* **JSON (JavaScript Object Notation):** The **language** the device and server speak. It is just a simple list of keys and values.
* *Example:* `{"sensor_id": "Room1", "temp": 22.5}`


* **Endpoint:** A specific **URL** where your API lives.
* *Example:* `http://localhost/my_project/create.php`



---

### 3. The 4 Golden Rules: CRUD

Every database application does four things. We call this **CRUD**:

| Action | Meaning | HTTP Method | IoT Example |
| --- | --- | --- | --- |
| **C**reate | Add new data | **POST** | Sending a new temp reading. |
| **R**ead | View data | **GET** | Checking the history on a dashboard. |
| **U**pdate | Edit data | **PUT/POST** | Changing a sensor's name. |
| **D**elete | Remove data | **DELETE/POST** | Deleting old logs to save space. |

---

### 4. Why Security Matters (SQL Injection)

You never trust the data coming from the internet. A hacker could send a "command" instead of a "temperature" to delete your whole database.

* **The Fix:** We use **Prepared Statements**. Instead of putting data directly into the query, we use a `?` as a placeholder. This tells the database: *"Treat this input only as text, not as a command."*

---

### 💡 Summary for the Lab

* **PHP** is the brain (Logic).
* **MySQL** is the memory (Storage).
* **Postman** is our "Fake Sensor" used to test if our brain and memory are working correctly.

---

===
===
===
===
---














# Practice Part
# 📖 Instructor's Guide: IoT API Development

## Module NITIW401: Backend Integration

This document contains the fully commented source code and testing procedures for students.

---

## 🛠 1. Database Connection (`db.php`)

**Purpose:** To establish a secure link between the PHP script and the MySQL database.

```php
<?php
// Define the address of the database server (usually localhost for XAMPP)
$host = 'localhost'; 

// The default username for XAMPP MySQL is 'root'
$user = 'root'; 

// The default password for XAMPP is empty/blank
$pass = ''; 

// The name of the database we created in phpMyAdmin
$db   = 'iot_database'; 

// Create a new connection object using the MySQLi library
$conn = new mysqli($host, $user, $pass, $db); 

// Check if the connection failed
if ($conn->connect_error) {
    // If there is an error, stop the script and display the message
    die("Connection failed: " . $conn->connect_error); 
}
?>

```

---

## 🚀 2. CRUD Operations

### 🟢 CREATE Data (`create.php`)

**Purpose:** Allows the IoT device to send sensor readings to the database.

```php
<?php
// Include the database connection settings from db.php
include 'db.php'; 

// Get the raw text (JSON) sent by the device/Postman
$jsonInput = file_get_contents('php://input'); 

// Convert the JSON text into a PHP 'Array' so we can work with it
$data = json_decode($jsonInput, true); 

// Validation: Check if the required fields (sensor_id and temperature) exist
if (!isset($data['sensor_id']) || !isset($data['temperature'])) {
    // If data is missing, tell the device it made a 'Bad Request' (Error 400)
    http_response_code(400);
    echo json_encode(["error" => "Invalid data. Missing ID or Temp."]); 
    exit; // Stop the script here
}

// Assign the data from the array to simple variables
$sensor_id = $data['sensor_id'];
$temperature = $data['temperature'];
$humidity = $data['humidity'];

// Prepare an SQL statement with '?' placeholders to prevent SQL Injection (hacking)
$sql = "INSERT INTO sensor_data (sensor_id, temperature, humidity) VALUES (?, ?, ?)";

// Tell the database to prepare for this specific query structure
$stmt = $conn->prepare($sql); 

// Bind the actual data to the '?' placeholders
// "sdd" means: s = string, d = double (decimal), d = double
$stmt->bind_param("sdd", $sensor_id, $temperature, $humidity); 

// Run the query and check if it was successful
if ($stmt->execute()) {
    // Send a success message back to the device in JSON format
    echo json_encode(["message" => "Data inserted successfully."]);
} else {
    // If something went wrong, send the error message
    echo json_encode(["error" => "Error: " . $stmt->error]); 
}

// Close the statement and the database connection to save server resources
$stmt->close();
$conn->close();
?>

```

---

### 🔵 READ Data (`read.php`)

**Purpose:** Retrieves all stored sensor data for display.

```php
<?php
// Include the connection file
include 'db.php'; 

// Create the SQL command to select everything and sort by the latest date
$sql = "SELECT * FROM sensor_data ORDER BY created_at DESC"; 

// Execute the query and store the result in a variable
$result = $conn->query($sql); 

// Check if the database actually returned any rows
if ($result->num_rows > 0) {
    // Create an empty list (array) to store the data
    $rows = array(); 
    
    // Fetch each row one by one as an associative array
    while ($row = $result->fetch_assoc()) {
        // Add the current row to our list
        $rows[] = $row; 
    }
    
    // Convert the entire list of rows into a JSON string for the user to see
    echo json_encode($rows);
} else {
    // If the table is empty, send a 'No data' message
    echo json_encode(["message" => "No data found."]); 
}

// Close the connection
$conn->close();
?>

```

---

### 🟡 UPDATE Data (`update.php`)

**Purpose:** Allows the system to modify existing sensor records (e.g., correcting an error).

```php
<?php
// Include the database connection bridge
include 'db.php'; 

// Grab the JSON data sent in the request body
$jsonInput = file_get_contents('php://input');
$data = json_decode($jsonInput, true);

// 1. Extract the ID and new values from the array
// We need the 'id' to know exactly which row to change
$id = $data['id']; 
$new_temp = $data['temperature'];
$new_hum = $data['humidity'];

// 2. Prepare the UPDATE command
// WHERE id = ? is critical! Without it, EVERY row in your database would change.
$sql = "UPDATE sensor_data SET temperature = ?, humidity = ? WHERE id = ?"; 

$stmt = $conn->prepare($sql);

// 3. Bind parameters: d = double (temp), d = double (humidity), i = integer (id)
$stmt->bind_param("ddi", $new_temp, $new_hum, $id); 

// 4. Execute and check for success
if ($stmt->execute()) {
    echo json_encode(["message" => "Record updated successfully."]); 
} else {
    echo json_encode(["error" => "Error updating record: " . $stmt->error]);
}

// 5. Close resources
$stmt->close();
$conn->close();
?>

```

---

### 🔴 DELETE Data (`delete.php`)

**Purpose:** Removes a specific record from the database using its unique ID.

```php
<?php
// Include the database connection bridge
include 'db.php'; 

// Capture the JSON input containing the ID to be deleted
$jsonInput = file_get_contents('php://input');
$data = json_decode($jsonInput, true);
$id = $data['id'];

// 1. Prepare the DELETE command
// Again, we use WHERE id = ? to ensure we only delete one specific row
$sql = "DELETE FROM sensor_data WHERE id = ?"; 

$stmt = $conn->prepare($sql);

// 2. Bind the ID as an integer (i)
$stmt->bind_param("i", $id); 

// 3. Execute the command
if ($stmt->execute()) {
    // Check if a row was actually removed (affected)
    if ($stmt->affected_rows > 0) {
        echo json_encode(["message" => "Record deleted successfully."]); 
    } else {
        // This happens if the ID provided doesn't exist in the database
        echo json_encode(["error" => "No record found with that ID."]); 
    }
} else {
    echo json_encode(["error" => "Error deleting record."]);
}

// 4. Close resources
$stmt->close();
$conn->close();
?>

```

---

## 🛠️ 4. Troubleshooting Guide (For Students)

If your code isn't working, check these common "Gotchas" before asking for help:

| Error Message / Issue | Common Cause | The Fix |
| --- | --- | --- |
| **"Connection failed: Access denied..."** | Wrong database password or username. | Check `db.php`. In XAMPP, user is `root` and pass is `''` (empty). |
| **"404 Not Found" in Postman** | The URL path is typed incorrectly. | Double-check your folder names and file names (e.g., `create.php` vs `Create.php`). |
| **"500 Internal Server Error"** | There is a syntax error in your PHP code. | Check for missing semicolons `;` or missing curly braces `{}`. |
| **Empty result `[]` in Read** | The database table is empty. | Run your **Create** test in Postman first to add some data. |
| **JSON returns "null"** | The data sent in Postman isn't valid JSON. | Ensure your JSON has double quotes `" "` and no trailing commas. |

---





---

## 🧪 3. Student Testing Guide (Postman)

To verify the code is working correctly, students should follow these steps for each "Endpoint."

### **Test A: The CREATE Endpoint**

1. **Method:** `POST`
2. **URL:** `http://localhost/api/create.php`
3. **Body:** Select **raw** and set type to **JSON**.
4. **Data to send:**

```json
{
    "sensor_id": "Lab_Sensor_01",
    "temperature": 24.5,
    "humidity": 55.0
}

```

### **Test B: The READ Endpoint**

1. **Method:** `GET`
2. **URL:** `http://localhost/api/read.php`
3. **Body:** None (Keep it empty).
4. **Expected Result:** You should see the JSON data you just "Created" in Test A.

### **Test C: The UPDATE Endpoint**

1. **Method:** `POST`
2. **URL:** `http://localhost/api/update.php`
3. **Data to send (Change ID to match a real row):**

```json
{
    "id": 1,
    "temperature": 99.9,
    "humidity": 10.0
}

```

### **Test D: The DELETE Endpoint**

1. **Method:** `POST`
2. **URL:** `http://localhost/api/delete.php`
3. **Data to send:**

```json
{
    "id": 1
}

```