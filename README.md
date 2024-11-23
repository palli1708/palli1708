import mysql.connector
from datetime import datetime

# Database Connection Setup
def connect_db():
    connection = mysql.connector.connect(
        host="localhost",  # MySQL server address
        user="root",       # MySQL username
        password="12345",  # MySQL password
        database="HotelDB" # Database name
    )
    return connection

# Room Class (Updated for DB interaction)
class Room:
    def __init__(self, number, room_type, price, is_occupied=False, customer_name=None, check_in_date=None, check_out_date=None):
        self.number = number
        self.room_type = room_type
        self.price = price
        self.is_occupied = is_occupied
        self.customer_name = customer_name
        self.check_in_date = check_in_date
        self.check_out_date = check_out_date

    def save(self):
        connection = connect_db()
        cursor = connection.cursor()
        cursor.execute(""" 
            INSERT INTO rooms (number, room_type, price, is_occupied, customer_name, check_in_date, check_out_date)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            room_type = %s, price = %s, is_occupied = %s, customer_name = %s, check_in_date = %s, check_out_date = %s
        """, (self.number, self.room_type, self.price, self.is_occupied, self.customer_name, self.check_in_date, self.check_out_date,
              self.room_type, self.price, self.is_occupied, self.customer_name, self.check_in_date, self.check_out_date))
        connection.commit()
        connection.close()

    @classmethod
    def fetch_all(cls):
        connection = connect_db()
        cursor = connection.cursor()
        cursor.execute("SELECT * FROM rooms")
        rooms = cursor.fetchall()
        connection.close()
        return [cls(*room) for room in rooms]

    @classmethod
    def fetch_available(cls):
        connection = connect_db()
        cursor = connection.cursor()
        cursor.execute("SELECT * FROM rooms WHERE is_occupied = 0")  # Only fetch available rooms
        rooms = cursor.fetchall()
        connection.close()
        return [cls(*room) for room in rooms]

# Customer Class (Updated for DB interaction)
class Customer:
    def __init__(self, name, room_number, check_in_date, check_out_date):
        self.name = name
        self.room_number = room_number
        self.check_in_date = check_in_date
        self.check_out_date = check_out_date

    def save(self):
        connection = connect_db()
        cursor = connection.cursor()
        cursor.execute(""" 
            INSERT INTO customers (name, room_number, check_in, check_out)
            VALUES (%s, %s, %s, %s)
        """, (self.name, self.room_number, self.check_in_date, self.check_out_date))
        connection.commit()
        connection.close()

    @classmethod
    def fetch_all(cls):
        connection = connect_db()
        cursor = connection.cursor()
        cursor.execute("SELECT * FROM customers")
        customers = cursor.fetchall()
        connection.close()
        return [cls(customer[1], customer[2], customer[3], customer[4]) for customer in customers]

# Hotel Class (Updated with DB functionality)
class Hotel:
    def __init__(self, name):
        self.name = name
        self.rooms = Room.fetch_all()  # Fetch rooms from DB
        self.customers = Customer.fetch_all()  # Fetch customers from DB
        self.add_default_rooms()  # Add default rooms if not present
        self.add_sample_customers()  # Add sample customers if not present

    def add_default_rooms(self):
        # Insert default rooms into the database if they don't exist
        default_rooms = [
            (101, "Deluxe", 1000),
            (102, "Standard", 800),
            (103, "Deluxe", 1000),
            (104, "Standard", 800),
            (105, "Deluxe", 1000)
        ]
        connection = connect_db()
        cursor = connection.cursor()
        for room in default_rooms:
            cursor.execute("""
                INSERT IGNORE INTO rooms (number, room_type, price) 
                VALUES (%s, %s, %s)
            """, room)
        connection.commit()
        connection.close()

    def add_sample_customers(self):
        # Insert sample customers into the database if they don't exist
        sample_customers = [
            ("John Doe", 101, "2024-11-01", "2024-11-05"),
            ("Jane Smith", 102, "2024-11-10", "2024-11-15")
        ]
        connection = connect_db()
        cursor = connection.cursor()
        for customer in sample_customers:
            cursor.execute("""
                INSERT IGNORE INTO customers (name, room_number, check_in, check_out) 
                VALUES (%s, %s, %s, %s)
            """, customer)
        connection.commit()
        connection.close()

    def display_rooms(self, only_available=False):
        available_rooms = Room.fetch_available() if only_available else self.rooms
        print("Available Rooms:" if only_available else "All Rooms:")
        for room in available_rooms:
            print(f"Room Number: {room.number}, Room Type: {room.room_type}, Price: {room.price}, "
                  f"Status: {'Occupied' if room.is_occupied else 'Available'}")

    def book_room(self, customer_name, room_number, check_in_date, check_out_date):
        check_in_date = datetime.strptime(check_in_date, "%Y-%m-%d")
        check_out_date = datetime.strptime(check_out_date, "%Y-%m-%d")
        
        # Find the room to book
        room_to_book = None
        for room in self.rooms:
            if room.number == room_number and not room.is_occupied:
                room_to_book = room
                break

        if room_to_book:
            # Update the room status to booked
            room_to_book.is_occupied = True
            room_to_book.customer_name = customer_name
            room_to_book.check_in_date = check_in_date
            room_to_book.check_out_date = check_out_date
            room_to_book.save()  # Save room changes to DB

            # Create and save the customer booking
            customer = Customer(customer_name, room_number, check_in_date, check_out_date)
            customer.save()  # Save customer booking to DB
            print(f"\nRoom {room_number} booked successfully for {customer_name} from {check_in_date} to {check_out_date}.")
        else:
            print("Room not available or already booked.")

    def cancel_booking(self, room_number):
        for room in self.rooms:
            if room.number == room_number and room.is_occupied:
                room.is_occupied = False
                room.customer_name = None
                room.check_in_date = None
                room.check_out_date = None
                room.save()  # Save room changes to DB
                print(f"Booking for Room {room_number} canceled.")
                return
        print("Room not booked or already canceled.")

    def display_customers(self):
        print("Customer Details and Booking History:")
        if not self.customers:
            print("No customers found.")
        for customer in self.customers:
            print(f"Customer Name: {customer.name}, Room: {customer.room_number}, "
                  f"Check-in: {customer.check_in_date}, Check-out: {customer.check_out_date}")

    def generate_bill(self, room_number):
        for room in self.rooms:
            if room.number == room_number and room.is_occupied:
                days = (room.check_out_date - room.check_in_date).days
                total_cost = days * room.price
                print(f"Bill for Room {room_number}: \nCustomer: {room.customer_name}, "
                      f"Total Stay: {days} days, Total Amount: ${total_cost}")
                return
        print(f"No booking found for room {room_number}.")

    def hotel_statistics(self):
        total_rooms = len(self.rooms)
        available_rooms = len([room for room in self.rooms if not room.is_occupied])
        total_earnings = sum([(room.check_out_date - room.check_in_date).days * room.price for room in self.rooms if room.is_occupied])

        print(f"Total Rooms: {total_rooms}")
        print(f"Available Rooms: {available_rooms}")
        print(f"Total Earnings: ${total_earnings}")

# Main Loop
hotel = Hotel("Hotel Heritage")

while True:
    print("\n=============Hotel Management System=============\n")
    print("1. Display available rooms")
    print("2. Book a room")
    print("3. Cancel a booking")
    print("4. View customer details")
    print("5. Generate bill")
    print("6. Hotel statistics")
    print("7. Exit\n")
    print("===================================================\n")
    choice = int(input("Enter your choice: "))

    if choice == 1:
        available = input("Show only available rooms? (yes/no): ").lower() == "yes"
        hotel.display_rooms(only_available=available)
    elif choice == 2:
        customer_name = input("Enter customer name: ")
        room_number = int(input("Enter room number: "))
        check_in_date = input("Enter check-in date (YYYY-MM-DD): ")
        check_out_date = input("Enter check-out date (YYYY-MM-DD): ")
        hotel.book_room(customer_name, room_number, check_in_date, check_out_date)
    elif choice == 3:
        room_number = int(input("Enter room number: "))
        hotel.cancel_booking(room_number)
    elif choice == 4:
        hotel.display_customers()
    elif choice == 5:
        room_number = int(input("Enter room number: "))
        hotel.generate_bill(room_number)
    elif choice == 6:
        hotel.hotel_statistics()
    elif choice == 7:
        confirm_exit = input("Are you sure you want to exit? (yes/no): ").lower()
        if confirm_exit == "yes":
            print("******Thank you for using our services!!******")
            break
    else:
        print("Invalid choice. Please try again.")

