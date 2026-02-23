    Flight Table (Main Table)
Purpose: Stores flight details.
Attributes:
Flight_ID (PK)
Flight_Number
Aircraft_ID (FK)
Route_ID (FK)
Departure_Date
Departure_Time
Arrival_Time
Status (Scheduled / Delayed / Cancelled / Completed)
2️⃣ Aircraft Table
Purpose: Stores aircraft details.
Attributes:
Aircraft_ID (PK)
Aircraft_Model
Capacity
Manufacturer
Maintenance_Status
🔗 Relation:
One Aircraft → Many Flights
3️⃣ Airport Table
Purpose: Stores airport details.
Attributes:
Airport_Code (PK)
Airport_Name
City
Country
🔗 Used in Route table.
4️⃣ Route Table
Purpose: Defines departure & arrival airports.
Attributes:
Route_ID (PK)
Departure_Airport (FK → Airport_Code)
Arrival_Airport (FK → Airport_Code)
Distance
🔗 One Route → Many Flights
5️⃣ Crew Table
Purpose: Stores crew details.
Attributes:
Crew_ID (PK)
Name
Role (Pilot / Co-Pilot / Cabin Crew)
Contact
License_Number
6️⃣ Flight_Crew Table (Junction Table)
Purpose: Assign crew to flights.
Attributes:
Flight_ID (FK)
Crew_ID (FK)
(Composite Primary Key: Flight_ID + Crew_ID)
🔗 Many-to-Many Relationship:
One Flight → Many Crew
One Crew → Many Flights
7️⃣ Maintenance Table (Very Good for Operations Module)
Purpose: Track aircraft maintenance.
Attributes:
Maintenance_ID (PK)
Aircraft_ID (FK)
Maintenance_Date
Maintenance_Type
Status
Technician_Name
🔗 Final Relationship Structure (For Viva)
Aircraft → Flight (1:M)
Route → Flight (1:M)
Airport → Route (1:M)
Flight ↔ Crew (M:M via Flight_Crew)
Aircraft → Maintenance (1:M)
