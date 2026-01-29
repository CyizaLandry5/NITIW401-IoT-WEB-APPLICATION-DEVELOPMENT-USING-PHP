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