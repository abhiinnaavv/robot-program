# robot-program
import socket
import subprocess
import time
import json
from PyQt5.QtCore import QThread, pyqtSignal, Qt
from PyQt5.QtGui import QFont, QPixmap
from PyQt5.QtWidgets import QApplication, QMainWindow, QPushButton, QLabel, QVBoxLayout, QWidget

class ClientThread(QThread):
    status_signal = pyqtSignal(str)
    machine1_signal = pyqtSignal(str)
    machine2_signal = pyqtSignal(str)
    action_signal = pyqtSignal(str)
    robot_signal = pyqtSignal(bool)
    machine_signal_glow = pyqtSignal(bool)
    model_signal = pyqtSignal(str)
    gripper_signal = pyqtSignal(str)
    start_machine_received_signal = pyqtSignal(int)  # Add signal for received count
    start_machine_sent_signal = pyqtSignal(int)      # Add signal for sent count


    def __init__(self, server_ip, server_port, model_file, parent=None):
        super().__init__(parent)
        self.server_ip = server_ip
        self.server_port = server_port
        self.socket = None 
        self.running = False
        self.connected = False
        self.last_sent_command = None
        self.start_command_sent = False
        self.startmachine_received = False
        self.retry_count = 0
        self.reconnect_on_next_round = False
        self.model_file = model_file
        self.model_index = 0
        self.current_row = 0  # Track which row to update in Excel
        self.program_resumed = False
        self.program_paused = False
        self.machine_status = ""
        self.m1_current_status = ""
        self.m2_current_status = "" 
        self.m1_previous_status = ""
        self.m2_previous_status = ""
        self.m2_contain_phone = False
        self.buffer_contain_phone = False
        # Flags to control when to emit model and gripper status
        self.update_model_status = False
        self.update_gripper_status = False
        self.robot_request = False
        self.operator_contain_phone = True
        # Counters to track command receptions and sends
        self.start_machine_received_count = 0
        self.start_machine_sent_count = 0
        self.m2_result = ""
        self.m1_result = ""

    def run(self):
        while self.running:
            if not self.connected:
                self.connect_to_server()
            else:
                try:
                    while self.connected and self.running:
                        # Dynamically check the machine status in every cycle
                        if self.robot_request == True: 
                            self.robot_request = False
                            self.current_m1_status = self.check_machine1_status()
                            self.current_m2_status = self.check_machine2_status()
                            print(self.current_m1_status)
                            print(self.current_m2_status)   
                            # Handle actions based on machine status
                            if self.current_m2_status == "IDLE" and self.m2_contain_phone == False and self.buffer_contain_phone == True:
                                self.m2_contain_phone = True
                                self.buffer_contain_phone = False
                                self.send_robot("BufferToM2")
                            elif self.current_m1_status == "READY" and self.m2_contain_phone == True and self.buffer_contain_phone == False: 
                                if self.m1_result == "error":
                                    self.send_robot("M1ToError")
                                else:
                                    self.buffer_contain_phone = True
                                    self.send_robot("M1ToBuffer")
                            elif self.current_m1_status == "READY" and self.current_m2_status == "IDLE" and self.m2_contain_phone == False and self.buffer_contain_phone == False:
                                if self.m1_result == "error":
                                    self.send_robot("M1ToError")
                                else:
                                    self.m2_contain_phone = True
                                    self.send_robot("M1ToM2")
                            elif self.current_m2_status == "IDLE" and self.m2_contain_phone == True:
                                self.m2_contain_phone = False
                                if self.m2_result == "error":
                                    self.send_robot("M2ToError")
                                else:
                                    self.send_robot("M2ToTested")
                            elif self.current_m1_status == "IDLE":
                                self.send_robot("OperatorToM1")
                            else:   
                                print("No command")
                                self.send_robot("NOcommand")
                        server_response = self.socket.recv(1024)
                        if server_response:
                            self.robot_signal.emit(True)
                            self.status_signal.emit(f"Server Response: {server_response.decode()}")
                            self.retry_count = 0
                            self.handle_received_message(server_response.decode().strip())
                        time.sleep(1)
                except socket.error as e:
                    print("Connection error:", e)
                    self.connected = False
                    self.robot_signal.emit(False)
                    self.status_signal.emit(f"Connection error: {e}. Retrying...")
                    if self.socket:
                        self.socket.close()
                    self.connect_to_server()
                    self.reconnect_on_next_round = True

    def connect_to_server(self):
        self.status_signal.emit("Attempting to connect to server...")
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.settimeout(500)
        try:
            self.socket.connect((self.server_ip, self.server_port))
            self.connected = True
            self.robot_signal.emit(True)
            self.status_signal.emit("Connected to server!")
        except socket.error as e:
            self.robot_signal.emit(False)
            self.status_signal.emit(f"Connection failed: {e}. Retrying...")
            self.connected = False

    def handle_received_message(self, message):
        """
        Handles the received message and initiates the machine start command or other actions.
        """
        print(f"Received message: {message}")  # Debugging
        if message == "Request":
            print("Request received")
            self.robot_request = True
        elif message == "startM2":
            print("StartMachine received")
            self.startmachine_received = True   
            self.send_machine_start_command()
    def send_machine_start_command(self):
        """
        This function sends the machine start command to the server using curl.
        """
        print("Sending Machine Start Command...")  # Debugging
        self.status_signal.emit("Sending Machine Start Command...")
        
        try:
            # Send start command via curl
            response = subprocess.run(
                [
                    'curl', '--request', 'POST',
                    'http://10.11.157.94:8084/automation-api/start', 
                    '-H', 'Content-Type: application/json',
                    '-H', 'Content-Length: 0'
                    # NO '-d' parameter at all
                ],
                capture_output=True, text=True
            )

            print(f"Response: {response.stdout}, Error: {response.stderr}, Return Code: {response.returncode}")

            if response.returncode == 0:
                print("Machine Start Command sent successfully.")  # Debugging
                self.start_machine_sent_count += 1  # Increment the sent counter
                self.status_signal.emit("Machine Start Command sent successfully.")
                self.status_signal.emit(f"Failed to send Machine Start Command: {response.stderr}")
                self.start_machine_sent_signal.emit(self.start_machine_sent_count) 
                
 

        except Exception as e:
            print(f"Error sending start command: {e}")
            self.status_signal.emit(f"Error sending start command: {e}")

    def send_robot(self, command):
        print("Sending command to robot...")
        print(command)
        self.socket.sendall(command.encode())
        self.status_signal.emit(f"Sent: {command}")
        self.action_signal.emit(f"Sent: {command}")
    def check_machine1_status(self):
        try:
            response = subprocess.run(
                ['curl', '--request', 'GET', '  '],
                capture_output=True, text=True
            )
            status = response.stdout.strip()
            parsed_status = status
            data = json.loads(parsed_status)
            self.m1_result = data.get("result")
            print("machine 1 result")
            print(self.m1_result)
            self.machine1_signal.emit(f"Machine Status: {status}")
            # Return 'IDLE' or 'Non-IDLE' based on the actual machine status
            if "IDLE" in status.upper():  # This will ignore any other status
                return "IDLE"
            elif "TESTING" in status.upper():
                return "TESTING"
            elif "READY" in status.upper():
                return "READY"
            else:
                print(f"Machine is not IDLE, detected status: {status}")  # Debugging
                return "RUNNING"  # Non-IDLE status (e.g., running, error, etc.)
        except Exception as e:
            print(f"Error checking machine status: {e}")  # Debugging
            self.machine1_signal.emit(f"Error checking machine status: {e}")
            return "ERROR"

    def check_machine2_status(self):
        try:
            response = subprocess.run(
                ['curl', '--request', 'GET', 'http://10.11.157.94:8084/automation-api/status'],
                capture_output=True, text=True
            )
            status = response.stdout.strip()
            parsed_status = status
            data = json.loads(parsed_status)
            self.m2_result = data.get("result")
            print("machine 2 result")
            print(self.m2_result)
            self.machine2_signal.emit(f"Machine Status: {status}")
            # Return 'IDLE' or 'Non-IDLE' based on the actual machine status
            if "IDLE" in status.upper():  # This will ignore any other status
                return "IDLE"
            elif "TESTING" in status.upper():
                return "TESTING"
            elif "READY" in status.upper():
                return "READY"
            else:
                print(f"Machine is not IDLE, detected status: {status}")  # Debugging
                return "RUNNING"  # Non-IDLE status (e.g., running, error, etc.)
        except Exception as e:
            print(f"Error checking machine status: {e}")  # Debugging
            self.machine2_signal.emit(f"Error checking machine status: {e}")
            return "ERROR"

    def reset_commands(self):
        """
        Resets flags for the next cycle so the program can process new commands
        without being blocked by previous cycle state.
        """
        self.startmachine_received = False
        self.start_command_sent = False
        self.program_resumed = False
        self.program_paused = False

    def stop(self):
        self.running = False
        if self.socket:
            self.socket.close()


class ClientApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Robot Control HMI")
        self.setGeometry(100, 100, 600, 500)
        self.font = QFont("Arial", 14)

        # Thread Initialization
        self.thread = ClientThread('192.168.58.2', 2111, "/users/macos/Downloads/Grippclient 2 - Copy.xlsx")
        self.thread.status_signal.connect(self.update_status)
        self.thread.machine1_signal.connect(self.update_machine1_status)
        self.thread.machine2_signal.connect(self.update_machine2_status)
        self.thread.action_signal.connect(self.update_action_status)
        self.thread.robot_signal.connect(self.update_robot_glow)
        self.thread.machine_signal_glow.connect(self.update_machine_glow)
        self.thread.model_signal.connect(self.update_model_display)
        self.thread.gripper_signal.connect(self.update_gripper_status_display)
        self.thread.start_machine_received_signal.connect(self.update_start_machine_received_count)
        self.thread.start_machine_sent_signal.connect(self.update_start_machine_sent_count)

        self.init_ui()

    def init_ui(self):
        self.logo_label = QLabel(self)
        pixmap = QPixmap("Logo.svg")
        pixmap = pixmap.scaled(200, 200, Qt.KeepAspectRatio)
        self.logo_label.setPixmap(pixmap)
        self.logo_label.setAlignment(Qt.AlignCenter)

        self.status_label = QLabel("Status: Not Connected", self)
        self.status_label.setFont(self.font)

        self.machine1_status_label = QLabel("Machine Status 1: Unknown", self)
        self.machine1_status_label.setFont(self.font)

        self.machine2_status_label = QLabel("Machine Status 2: Unknown", self)
        self.machine2_status_label.setFont(self.font)

        self.action_label = QLabel("Action: None", self)
        self.action_label.setFont(self.font)

        self.gripper_label = QLabel("Gripper: None", self)
        self.gripper_label.setFont(self.font)

        self.model_label = QLabel("Model: None", self)
        self.model_label.setFont(self.font)

        # Labels for tracking start machine commands
        self.start_machine_received_label = QLabel("Start Machine Received: 0", self)
        self.start_machine_received_label.setFont(self.font)

        self.start_machine_sent_label = QLabel("Start Machine Sent: 0", self)
        self.start_machine_sent_label.setFont(self.font)

        self.robot_button = QPushButton("Robot")
        self.robot_button.setStyleSheet("background-color: #B0BEC5; color: white; border-radius: 5px; padding: 15px;")

        self.client_button = QPushButton("Client")
        self.client_button.setStyleSheet("background-color: #B0BEC5; color: white; border-radius: 5px; padding: 15px;")

        self.machine_button = QPushButton("Machine")
        self.machine_button.setStyleSheet("background-color: #B0BEC5; color: white; border-radius: 5px; padding: 15px;")

        self.start_button = QPushButton("Start Listening")
        self.start_button.setStyleSheet("background-color: #007BFF; color: white; padding: 10px;")
        self.start_button.clicked.connect(self.start_listening)

        self.stop_button = QPushButton("Stop Listening")
        self.stop_button.setStyleSheet("background-color: #FF5733; color: white; padding: 10px;")
        self.stop_button.clicked.connect(self.stop_listening)

        layout = QVBoxLayout()
        layout.addWidget(self.logo_label)
        layout.addWidget(self.status_label)
        layout.addWidget(self.machine1_status_label)
        layout.addWidget(self.machine2_status_label)
        layout.addWidget(self.action_label)
        layout.addWidget(self.model_label)
        layout.addWidget(self.gripper_label)
        layout.addWidget(self.robot_button)
        layout.addWidget(self.machine_button)
        layout.addWidget(self.client_button)
        layout.addWidget(self.start_machine_received_label)
        layout.addWidget(self.start_machine_sent_label)
        layout.addWidget(self.start_button)
        layout.addWidget(self.stop_button)

        container = QWidget()
        container.setLayout(layout)
        self.setCentralWidget(container)

    def start_listening(self):
        if not self.thread.isRunning():
            self.thread.running = True
            self.thread.start()
            self.status_label.setText("Listening started.")
        else:
            self.status_label.setText("Already listening.")

    def stop_listening(self):
        self.thread.stop()
        self.status_label.setText("Status: Listening stopped.")

    def update_status(self, message):
        self.status_label.setText(f"Status: {message}")

    def update_machine1_status(self, machine_status):
        self.machine1_status_label.setText(f"Machine Status 1: {machine_status}")

    def update_machine2_status(self, machine_status):
        self.machine2_status_label.setText(f"Machine Status 2: {machine_status}")

    def update_action_status(self, action_status):
        self.action_label.setText(f"Action: {action_status}")

    def update_robot_glow(self, glow):
        if glow:
            self.robot_button.setStyleSheet("background-color: #4CAF50; color: white; border-radius: 5px; padding: 15px;")
        else:
            self.robot_button.setStyleSheet("background-color: #B0BEC5; color: white; border-radius: 5px; padding: 15px;")

    def update_machine_glow(self, glow):
        if glow:
            self.machine_button.setStyleSheet("background-color: #4CAF50; color: white; border-radius: 5px; padding: 15px;")
        else:
            self.machine_button.setStyleSheet("background-color: #B0BEC5; color: white; border-radius: 5px; padding: 15px;")

    def update_model_display(self, model):
        self.model_label.setText(f"Model: {model}")

    def update_gripper_status_display(self, gripper_status):
        self.gripper_label.setText(f"Gripper: {gripper_status}")
        
    def update_start_machine_received_count(self, count):
        self.start_machine_received_label.setText(f"Start Machine Received: {count}")
        
    def update_start_machine_sent_count(self, count):
        self.start_machine_sent_label.setText(f"Start Machine Sent: {count}")


if __name__ == '__main__':
    app = QApplication([])
    window = ClientApp()
    window.show()
    app.exec_()
