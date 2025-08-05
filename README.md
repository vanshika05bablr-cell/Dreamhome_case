-- DreamHome Database Build Script for MBA Analytics Students
-- This script is designed for PostgreSQL and covers schema definition, data population,
-- and advanced database features like triggers for business rule enforcement.

-- =============================================================================
-- SECTION 1: DATABASE SCHEMA DEFINITION (CREATE TABLES)
-- This section defines the structure of each table, including primary keys,
-- data types, NOT NULL constraints, CHECK constraints, and foreign key relationships.
-- The order of table creation is crucial to satisfy foreign key dependencies.
-- =============================================================================

-- 1.1 Create Branch Table
-- Stores details about each office branch of DreamHome.
-- branchNo: Unique identifier for the branch (Primary Key).
-- street, city, postcode: Location details of the branch.
CREATE TABLE Branch (
    branchNo VARCHAR(4) PRIMARY KEY,
    street VARCHAR(50) NOT NULL,
    city VARCHAR(30) NOT NULL,
    postcode VARCHAR(10) NOT NULL
);

-- 1.2 Create Staff Table
-- Stores details about employees working at DreamHome branches.
-- staffNo: Unique identifier for the staff member (Primary Key).
-- fName, lName: First and last names of the staff member.
-- position: Role of the staff member (e.g., Manager, Assistant).
-- sex: Gender of the staff member (constrained to 'M' or 'F').
-- dob: Date of birth.
-- salary: Staff's salary (must be non-negative).
-- branchNo: Foreign Key linking staff to their respective branch.
CREATE TABLE Staff (
    staffNo VARCHAR(5) PRIMARY KEY,
    fName VARCHAR(30) NOT NULL,
    lName VARCHAR(30) NOT NULL,
    position VARCHAR(20) NOT NULL,
    sex CHAR(1) CHECK (sex IN ('M', 'F')),
    dob DATE,
    salary DECIMAL(10, 2) CHECK (salary >= 0),
    branchNo VARCHAR(4) NOT NULL,
    FOREIGN KEY (branchNo) REFERENCES Branch(branchNo)
);

-- Add currency column to Staff table (schema evolution)
-- This column was added after initial table creation, demonstrating ALTER TABLE.
-- currency: Stores the ISO 4217 three-letter currency code for the salary.
ALTER TABLE Staff
ADD COLUMN currency VARCHAR(3) NOT NULL DEFAULT 'AUD';


-- 1.3 Create PrivateOwner Table
-- Stores details about private individuals who own properties for rent.
-- ownerNo: Unique identifier for the owner (Primary Key).
-- fName, lName: First and last names of the owner.
-- address: (Deprecated - replaced by street, city, postcode)
-- telNo: Telephone number of the owner.
CREATE TABLE PrivateOwner (
    ownerNo VARCHAR(5) PRIMARY KEY,
    fName VARCHAR(30) NOT NULL,
    lName VARCHAR(30) NOT NULL,
    address VARCHAR(100) NOT NULL, -- Initial 'address' column
    telNo VARCHAR(15)
);

-- Refactoring PrivateOwner: Dropping single address column and adding structured address fields
-- This set of ALTER TABLE commands replaces a single 'address' field with more granular
-- 'street', 'city', and 'postcode' fields, improving data granularity and query flexibility.
ALTER TABLE PrivateOwner
DROP COLUMN address; -- Remove the old 'address' column

ALTER TABLE PrivateOwner
ADD COLUMN street VARCHAR(50) NOT NULL; -- Add new 'street' column

ALTER TABLE PrivateOwner
ADD COLUMN city VARCHAR(30) NOT NULL; -- Add new 'city' column

ALTER TABLE PrivateOwner
ADD COLUMN postcode VARCHAR(10) NOT NULL; -- Add new 'postcode' column


-- 1.4 Create PropertyForRent Table
-- Stores details about properties available for rent.
-- propertyNo: Unique identifier for the property (Primary Key).
-- street, city, postcode: Location details of the property.
-- type: Type of property (e.g., 'House', 'Flat'). Constrained by CHECK.
-- rooms: Number of rooms in the property (must be positive).
-- rent: Monthly rent amount (must be positive).
-- ownerNo: Foreign Key linking the property to its owner.
-- staffNo: Foreign Key linking the property to the staff member managing it (can be NULL if unassigned).
-- branchNo: Foreign Key linking the property to the branch managing it.
CREATE TABLE PropertyForRent (
    propertyNo VARCHAR(5) PRIMARY KEY,
    street VARCHAR(50) NOT NULL,
    city VARCHAR(30) NOT NULL,
    postcode VARCHAR(10) NOT NULL,
    type VARCHAR(20) CHECK (type IN ('House', 'Flat', 'Apartment', 'Bungalow')),
    rooms SMALLINT CHECK (rooms > 0),
    rent DECIMAL(10, 2) CHECK (rent > 0),
    ownerNo VARCHAR(5) NOT NULL,
    staffNo VARCHAR(5),
    branchNo VARCHAR(4) NOT NULL,
    FOREIGN KEY (ownerNo) REFERENCES PrivateOwner(ownerNo),
    FOREIGN KEY (staffNo) REFERENCES Staff(staffNo),
    FOREIGN KEY (branchNo) REFERENCES Branch(branchNo)
);


-- 1.5 Create Client Table
-- Stores details about clients who are looking to rent properties.
-- clientNo: Unique identifier for the client (Primary Key).
-- fName, lName: First and last names of the client.
-- telNo: Telephone number of the client.
-- prefType: Preferred type of property for rent (constrained by CHECK).
-- maxRent: Maximum rent the client is willing to pay (non-negative).
-- branchNo: Foreign Key linking the client to the branch where they registered.
CREATE TABLE Client (
    clientNo VARCHAR(5) PRIMARY KEY,
    fName VARCHAR(30) NOT NULL,
    lName VARCHAR(30) NOT NULL,
    telNo VARCHAR(15),
    prefType VARCHAR(20) CHECK (prefType IN ('House', 'Flat', 'Apartment', 'Bungalow')),
    maxRent DECIMAL(10, 2) CHECK (maxRent >= 0),
    branchNo VARCHAR(4) NOT NULL,
    FOREIGN KEY (branchNo) REFERENCES Branch(branchNo)
);

-- Add status column to Client table (schema evolution)
-- status: Indicates if the client is 'Open' (actively looking/renting) or 'Closed' (lease ended/no longer looking).
-- Default is 'Open'. Values are constrained by CHECK.
ALTER TABLE Client
ADD COLUMN status VARCHAR(10) NOT NULL DEFAULT 'Open'
CHECK (status IN ('Open', 'Closed'));


-- 1.6 Create Lease Table
-- Records details of lease agreements between clients and properties.
-- leaseNo: Auto-incrementing unique identifier for the lease (Primary Key, SERIAL).
-- clientNo: Foreign Key linking the lease to the client.
-- propertyNo: Foreign Key linking the lease to the property being leased.
-- rentStart, rentEnd: Start and end dates of the lease (rentEnd must be after rentStart).
-- rentAmount: Agreed weekly/monthly rent amount (must be positive).
-- paymentMethod: Method of payment for rent.
CREATE TABLE Lease (
    leaseNo SERIAL PRIMARY KEY,
    clientNo VARCHAR(5) NOT NULL,
    propertyNo VARCHAR(5) NOT NULL,
    rentStart DATE NOT NULL,
    rentEnd DATE NOT NULL,
    rentAmount DECIMAL(10, 2) NOT NULL CHECK (rentAmount > 0),
    paymentMethod VARCHAR(20),
    FOREIGN KEY (clientNo) REFERENCES Client(clientNo),
    FOREIGN KEY (propertyNo) REFERENCES PropertyForRent(propertyNo),
    CHECK (rentEnd > rentStart) -- Constraint: End date must be after start date
);


-- 1.7 Create Newspapers Table
-- Stores details about property advertisements placed in various newspapers.
-- newspaperAdNo: Auto-incrementing unique identifier for the advertisement (Primary Key, SERIAL).
-- newspaperName: Name of the newspaper.
-- street, city, postcode: Address of the newspaper office.
-- telephone: Contact number for the newspaper.
-- contactName: Contact person at the newspaper.
-- propertyNo: Foreign Key linking the advertisement to the advertised property.
-- dateAdvertised: Date when the advertisement was placed.
-- costToAdvertise: Cost incurred for the advertisement (non-negative).
CREATE TABLE Newspapers (
    newspaperAdNo SERIAL PRIMARY KEY,
    newspaperName VARCHAR(50) NOT NULL,
    street VARCHAR(50) NOT NULL,
    city VARCHAR(30) NOT NULL,
    postcode VARCHAR(10) NOT NULL,
    telephone VARCHAR(15),
    contactName VARCHAR(50) NOT NULL,
    propertyNo VARCHAR(5) NOT NULL,
    dateAdvertised DATE NOT NULL,
    costToAdvertise DECIMAL(10, 2) NOT NULL CHECK (costToAdvertise >= 0),
    FOREIGN KEY (propertyNo) REFERENCES PropertyForRent(propertyNo)
);


-- =============================================================================
-- SECTION 2: ADVANCED DATABASE FEATURES (TRIGGERS)
-- This section defines triggers to automate business rules.
-- Triggers ensure data consistency and automate state changes based on events.
-- =============================================================================

-- 2.1 Trigger Function: update_client_status_on_lease
-- This PL/pgSQL function is executed by a trigger to automatically
-- update a client's status to 'Closed' when they sign a new lease.
CREATE OR REPLACE FUNCTION update_client_status_on_lease()
RETURNS TRIGGER AS $$
BEGIN
    -- Update the status of the client associated with the newly inserted lease.
    -- Only change status if the client was previously 'Open' to avoid redundant updates.
    UPDATE Client
    SET status = 'Closed'
    WHERE clientNo = NEW.clientNo
    AND status = 'Open';

    RETURN NEW; -- For an AFTER INSERT trigger, NEW must be returned.
END;
$$ LANGUAGE plpgsql;

-- 2.2 Trigger Definition: client_status_update_trigger
-- This trigger fires AFTER an INSERT operation on the Lease table
-- for EACH ROW, executing the `update_client_status_on_lease` function.
CREATE TRIGGER client_status_update_trigger
AFTER INSERT ON Lease
FOR EACH ROW
EXECUTE FUNCTION update_client_status_on_lease();


-- =============================================================================
-- SECTION 3: DATA POPULATION (INSERT STATEMENTS)
-- This section populates the tables with sample data.
-- The order of insertions is critical due to foreign key constraints.
-- =============================================================================

-- 3.1 Insert Data into Branch Table
-- Creating branches around top universities in Melbourne and other key areas.
INSERT INTO Branch (branchNo, street, city, postcode) VALUES
('B001', 'University Plaza', 'Parkville', '3010'), -- Near University of Melbourne
('B002', 'Clayton Road', 'Clayton', '3168'),       -- Near Monash University (Clayton Campus)
('B003', 'Swanston Street', 'Melbourne CBD', '3000'), -- Near RMIT University (City Campus)
('B004', 'Burwood Highway', 'Burwood', '3125'),   -- Near Deakin University (Burwood Campus)
('B005', 'Bundoora Campus Dr', 'Bundoora', '3083'), -- Near La Trobe University (Bundoora Campus)
('B006', 'Hawthorn Campus Way', 'Hawthorn', '3122'), -- Near Swinburne University of Technology (Hawthorn Campus)
('B007', 'St Kilda Road', 'Southbank', '3006'),   -- Popular commercial area
('B008', 'Chapel Street', 'Prahran', '3181');     -- Fashion and retail hub


-- 3.2 Insert Data into Staff Table
-- Assigning multiple staff members to each branch, with varying positions and salaries.
-- Note: The 'currency' column will default to 'AUD' for these inserts as per the ALTER TABLE.
INSERT INTO Staff (staffNo, fName, lName, position, sex, dob, salary, branchNo) VALUES
('S001', 'Alice', 'Smith', 'Manager', 'F', '1985-03-15', 65000.00, 'B001'),
('S002', 'Bob', 'Johnson', 'Assistant', 'M', '1992-11-22', 45000.00, 'B001'),
('S003', 'Clara', 'Davis', 'Senior Assistant', 'F', '1988-07-01', 52000.00, 'B001'),
('S004', 'David', 'Miller', 'Manager', 'M', '1980-01-10', 68000.00, 'B002'),
('S005', 'Eve', 'Wilson', 'Assistant', 'F', '1995-04-25', 46000.00, 'B002'),
('S006', 'Frank', 'Moore', 'Manager', 'M', '1983-09-05', 66000.00, 'B003'),
('S007', 'Grace', 'Taylor', 'Assistant', 'F', '1993-02-18', 44000.00, 'B003'),
('S008', 'Harry', 'Anderson', 'Manager', 'M', '1978-06-30', 70000.00, 'B004'),
('S009', 'Ivy', 'Thomas', 'Assistant', 'F', '1990-10-10', 47000.00, 'B004'),
('S010', 'Jack', 'Jackson', 'Manager', 'M', '1982-04-03', 67000.00, 'B005'),
('S011', 'Karen', 'White', 'Assistant', 'F', '1991-08-14', 45500.00, 'B005'),
('S012', 'Liam', 'Harris', 'Manager', 'M', '1975-12-01', 72000.00, 'B006'),
('S013', 'Mia', 'Martin', 'Assistant', 'F', '1994-01-20', 44500.00, 'B006'),
('S014', 'Noah', 'Thompson', 'Manager', 'M', '1986-05-09', 69000.00, 'B007'),
('S015', 'Olivia', 'Garcia', 'Assistant', 'F', '1996-03-08', 43000.00, 'B007'),
('S016', 'Peter', 'Martinez', 'Manager', 'M', '1979-02-28', 71000.00, 'B008'),
('S017', 'Quinn', 'Robinson', 'Assistant', 'F', '1997-09-12', 42500.00, 'B008');


-- 3.3 Insert Data into PrivateOwner Table
-- Creating fictional private owners with detailed address information.
INSERT INTO PrivateOwner (ownerNo, fName, lName, street, city, postcode, telNo) VALUES
('P001', 'John', 'Doe', '123 River Rd', 'Melbourne', '3000', '0412345678'),
('P002', 'Jane', 'Smith', '45 Chapel St', 'Prahran', '3181', '0498765432'),
('P003', 'Robert', 'Brown', '789 Beach Ave', 'St Kilda', '3182', '0455112233'),
('P004', 'Emily', 'Green', '10 University Cres', 'Bundoora', '3083', '0477889900'),
('P005', 'Michael', 'White', '33 King St', 'Melbourne', '3000', '0422334455');


-- 3.4 Insert Data into PropertyForRent Table
-- Populating properties linked to owners, staff, and branches.
INSERT INTO PropertyForRent (propertyNo, street, city, postcode, type, rooms, rent, ownerNo, staffNo, branchNo) VALUES
('PFR01', '10 Parkville St', 'Parkville', '3010', 'Flat', 2, 450.00, 'P001', 'S002', 'B001'), -- Flat near UniMelb
('PFR02', '25 Carlton Ave', 'Carlton', '3053', 'Apartment', 1, 380.00, 'P005', 'S003', 'B001'), -- Apt near UniMelb
('PFR03', '15 University Dr', 'Clayton', '3168', 'House', 3, 550.00, 'P002', 'S005', 'B002'), -- House near Monash
('PFR04', '8 Blackburn Rd', 'Clayton', '3168', 'Flat', 2, 400.00, 'P003', 'S004', 'B002'), -- Flat near Monash
('PFR05', '50 Elizabeth St', 'Melbourne CBD', '3000', 'Apartment', 1, 420.00, 'P004', 'S007', 'B003'), -- Apt near RMIT
('PFR06', '12 Flinders Lane', 'Melbourne CBD', '3000', 'Flat', 2, 500.00, 'P001', 'S006', 'B003'), -- Flat near RMIT
('PFR07', '20 Forest Hill Rd', 'Burwood', '3125', 'House', 4, 600.00, 'P002', 'S009', 'B004'); -- House near Deakin


-- 3.5 Insert Data into Client Table
-- Populating clients registered at various branches.
-- Note: The 'status' column will default to 'Open' for these inserts as per the ALTER TABLE.
INSERT INTO Client (clientNo, fName, lName, telNo, prefType, maxRent, branchNo) VALUES
('C001', 'Chris', 'Evans', '0411223344', 'Apartment', 400.00, 'B001'),
('C002', 'Diana', 'Prince', '0466778899', 'Flat', 500.00, 'B001'),
('C003', 'Edward', 'Norton', '0433221100', 'House', 600.00, 'B002'),
('C004', 'Fiona', 'Apple', '0488776655', 'Apartment', 450.00, 'B003'),
('C005', 'George', 'Clooney', '0410987654', 'House', 700.00, 'B004');


-- 3.6 Insert Data into Lease Table
-- Creating a sample lease agreement between an existing client and property.
-- This insert will trigger the 'client_status_update_trigger' to change client C001's status to 'Closed'.
INSERT INTO Lease (clientNo, propertyNo, rentStart, rentEnd, rentAmount, paymentMethod) VALUES
('C001', 'PFR01', '2024-08-01', '2025-07-31', 450.00, 'Direct Debit');


-- 3.7 Insert Data into Newspapers Table
-- Recording various property advertisements.
INSERT INTO Newspapers (newspaperName, street, city, postcode, telephone, contactName, propertyNo, dateAdvertised, costToAdvertise) VALUES
('Melbourne Daily', '100 News Ave', 'Melbourne', '3000', '0399887766', 'Sarah Connor', 'PFR01', '2024-07-10', 150.00),
('Local Star Herald', '55 Community Blvd', 'Clayton', '3168', '0398765432', 'John Wick', 'PFR03', '2024-07-05', 120.50),
('City Property News', '22 Business St', 'Melbourne CBD', '3000', '0391234567', 'Maria Hill', 'PFR05', '2024-07-12', 200.00);


-- =============================================================================
-- SECTION 4: UTILITY COMMANDS (FOR DEMONSTRATION & TESTING)
-- These commands are useful for querying the data or resetting specific tables.
-- =============================================================================

-- Query all data from a table (e.g., to verify insertions)
SELECT * FROM Branch;
SELECT * FROM Staff;
SELECT * FROM PrivateOwner;
SELECT * FROM PropertyForRent;
SELECT * FROM Client;
SELECT * FROM Lease;
SELECT * FROM Newspapers;

-- Command to clear all entries from the Lease table and reset its SERIAL sequence
-- Useful for re-running scenarios or starting with a clean Lease table.
-- TRUNCATE TABLE Lease RESTART IDENTITY;
