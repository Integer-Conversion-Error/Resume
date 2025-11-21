# Autotrader-like Web Application

This project aims to develop a comprehensive web application similar to Autotrader, providing a platform for users to buy and sell vehicles. It features distinct user roles, a robust backend, a dynamic frontend, and a well-structured database. The application is designed to facilitate vehicle listings, search capabilities, user management, and future monetization features.

## 1. User Roles (Agents)

The platform will support the following user roles:

*   **Customer/Buyer:**
    *   **Description:** Individuals looking to find and purchase vehicles.
    *   **Key Actions:** Browse/search listings, view vehicle details, save favorites, contact sellers, manage profile.
*   **Private Seller:**
    *   **Description:** Individuals selling their personal vehicles.
    *   **Key Actions:** Register, create/manage listings (photos, descriptions, pricing), respond to buyers, manage profile.
*   **Dealer:**
    *   **Description:** Car dealerships or businesses selling multiple vehicles.
    *   **Key Actions:** Register dealership, manage business profile/storefront, create/manage large inventory (bulk uploads), advanced listing features, manage staff accounts, respond to inquiries, access analytics.
*   **Administrator:**
    *   **Description:** Platform owners/operators responsible for site management.
    *   **Key Actions:** Manage user accounts, oversee/moderate listings, manage platform settings, handle disputes, view site-wide analytics, manage subscriptions/fees, content management.
*   **(Optional) Guest User:**
    *   **Description:** Unregistered visitors.
    *   **Key Actions:** Browse/search listings with limited access, prompted to register for more actions.

## 2. Backend Technology

*   **Language/Framework:** Node.js with Express.js (or a similar modern framework like Fastify)
    *   Chosen for its event-driven, non-blocking I/O model, making it efficient for I/O-bound operations and real-time applications.
    *   Its vast npm ecosystem, large community, and the ability to use JavaScript/TypeScript across the full stack simplify development and accelerate time-to-market.

## 3. Frontend Technology

*   **Language/Framework:** React with Vite
    *   React is a popular JavaScript library for building user interfaces, known for its component-based architecture and large ecosystem.
    *   Vite provides a fast development server and build tool, significantly improving the developer experience for React projects.

## 4. Feature Prioritization

### MVP (Minimum Viable Product) Features:

*   **User Roles & Authentication:** Customer & Private Seller (registration, login, profile).
*   **Vehicle Listings (Private Seller Focus):** Create, edit, delete listings (make, model, year, VIN, price, mileage, description, images, contact, location).
*   **Vehicle Browsing & Searching (Customer Focus):** Basic search (make, model, year, price, location), list view, detail view.
*   **Communication:** Customers view seller contact info (external communication).
*   **Basic Admin:** Manual/simple interface to view/remove listings.

### Post-MVP Features:

*   Dealer Accounts (profiles, bulk listings, storefronts).
*   Advanced Search & Filtering (more criteria, sorting, saved searches, notifications).
*   In-App Messaging.
*   User Reviews & Ratings (dealers, private sellers).
*   Favorites/Watchlist.
*   Comprehensive Admin Dashboard (user management, moderation, analytics, settings).
*   Monetization Features (featured listings, subscriptions, listing fees).
*   Vehicle History Report Integration (e.g., Carfax).
*   Financing & Insurance Information/Links.
*   Mobile Apps (Native or PWA).

## 5. Key Features

### 5.1. Sponsored Listings Implementation Plan (V2 - Detailed)

This section outlines a comprehensive plan for implementing a sponsored listings feature. This feature allows users or dealerships to pay a fee to have their vehicle listings promoted for a set period, increasing their visibility.

#### 5.1.1. Core Concepts & User Flow

The implementation is based on a seamless user experience from purchase to promotion.

*   **User Flow**:
    1.  **Discovery**: A seller sees a "Sponsor this Listing" button on their "My Listings" page or on the public detail page for their vehicle.
    2.  **Package Selection**: Clicking the button opens a modal displaying the available `SponsorshipPackage` options, detailing the price, duration, and benefits (e.g., "Top of search," "Highlighted card").
    3.  **Payment**: The user selects a package and is redirected to a secure payment gateway (e.g., Stripe, PayPal) to complete the purchase.
    4.  **Confirmation**: After successful payment, the user is returned to the site, sees a confirmation message, and the listing is immediately marked as sponsored.
    5.  **Management**: In their dashboard, the user can see which of their listings are sponsored and when the sponsorship expires.

#### 5.1.2. Proposed Schema Changes

The database schema needs to be extended to support this feature.

##### `SponsorshipPackage` Table
This table defines the different sponsorship products you can offer.

```sql
CREATE TABLE SponsorshipPackage (
    package_id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL, -- e.g., 'Bronze Boost', 'Gold Feature'
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    duration_days INTEGER NOT NULL, -- How many days the sponsorship lasts
    display_priority INTEGER NOT NULL DEFAULT 0, -- Higher number means higher in search results
    highlight_color_hex VARCHAR(7), -- Optional: for special UI treatment
    is_active BOOLEAN DEFAULT TRUE -- Allows admins to enable/disable packages
);
```

##### `Transaction` Table
This table will store a record of all financial transactions on the platform.

```sql
CREATE TABLE Transaction (
    transaction_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES "User"(user_id) ON DELETE SET NULL,
    dealership_id INTEGER REFERENCES Dealership(dealership_id) ON DELETE SET NULL,
    listing_id INTEGER REFERENCES Listing(listing_id) ON DELETE SET NULL,
    package_id INTEGER REFERENCES SponsorshipPackage(package_id) ON DELETE SET NULL,
    product_description VARCHAR(255), -- e.g., 'Gold Sponsorship for 2023 Toyota Camry'
    amount DECIMAL(10, 2) NOT NULL,
    payment_gateway_charge_id VARCHAR(255) UNIQUE, -- e.g., Stripe or PayPal charge ID
    status VARCHAR(50) NOT NULL, -- e.g., 'succeeded', 'pending', 'failed'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

##### Modify `Listing` Table
We need to add columns to the `Listing` table to track the sponsorship status.

```sql
ALTER TABLE Listing
ADD COLUMN is_sponsored BOOLEAN NOT NULL DEFAULT FALSE,
ADD COLUMN sponsorship_package_id INTEGER REFERENCES SponsorshipPackage(package_id) ON DELETE SET NULL,
ADD COLUMN sponsorship_start_date TIMESTAMP WITH TIME ZONE,
ADD COLUMN sponsorship_end_date TIMESTAMP WITH TIME ZONE;
```

#### 5.1.3. Backend & API Logic

The backend will need several new endpoints to manage the process.

*   `GET /api/sponsorship-packages`: Returns a list of all active sponsorship packages for the UI.
*   `POST /api/listings/:id/create-sponsorship-checkout`:
    *   Takes a `package_id` in the request body.
    *   Creates a payment session with the payment gateway (e.g., Stripe Checkout).
    *   Returns a session ID or URL for the frontend to redirect the user to.
*   `POST /api/payment-webhook`:
    *   A secure endpoint that listens for webhook events from the payment gateway (e.g., `checkout.session.completed` from Stripe).
    *   **This is the most critical part.** Upon receiving a successful payment event, the webhook handler will:
        1.  Create a new record in the `Transaction` table.
        2.  Update the corresponding `Listing` record to set `is_sponsored` to `TRUE` and populate the sponsorship start/end dates.
        3.  Send a confirmation email to the user.

#### 5.1.4. Display & Sorting Logic

Sponsored listings must be given priority in the UI.

*   **Search Results**: When fetching vehicle listings, the primary sorting should be by sponsorship status and priority, then by the user's selected sort order (e.g., date, price).
    ```sql
    -- Example query ordering
    SELECT * FROM v_VehicleDetails
    ORDER BY
        is_sponsored DESC,       -- Sponsored listings first
        display_priority DESC,   -- Higher priority packages first
        listed_date DESC;        -- Then by default sorting
    ```
*   **UI Highlighting**: The frontend will use the `is_sponsored` flag and `highlight_color_hex` from the package to apply special styling to the vehicle card (e.g., a colored border, a "Sponsored" badge).

#### 5.1.5. Automated Expiry Management

A background job is needed to handle the expiration of sponsorships.

*   **Scheduled Task**: A cron job or other scheduler will run periodically (e.g., once every hour).
*   **Logic**: The job will execute a query to find all listings where `is_sponsored` is `TRUE` and the `sponsorship_end_date` is in the past.
    ```sql
    UPDATE Listing
    SET
        is_sponsored = FALSE,
        sponsorship_package_id = NULL,
        sponsorship_start_date = NULL,
        sponsorship_end_date = NULL
    WHERE
        is_sponsored = TRUE AND sponsorship_end_date < NOW();
    ```

#### 5.1.6. Analytics & Reporting

Tracking performance is key to proving value to sellers.

*   **Data to Track**: We can leverage the existing `vehicle_views` table. We should add a column to it to indicate if the view occurred while the listing was sponsored.
    ```sql
    ALTER TABLE vehicle_views ADD COLUMN was_sponsored BOOLEAN DEFAULT FALSE;
    ```
    When a view is recorded, the application logic will check the listing's `is_sponsored` status and set this flag accordingly.
*   **User Dashboard**: The seller's dashboard can then display analytics for their sponsored listings, such as:
    *   Total views while sponsored.
    *   A chart comparing views before and during the sponsorship period.
    *   Click-through rate (if tracking clicks on the listing card).

#### 5.1.7. Edge Case Considerations

*   **Listing Sold/Removed**: If a listing's status is changed to `SOLD` or `REMOVED`, the sponsorship should be immediately deactivated. This can be handled by application logic when the status is updated. Refunds are a business policy decision and not handled at the database level.
*   **Multiple Sponsorships**: The current model assumes a listing can only have one active sponsorship at a time. If a user tries to sponsor an already-sponsored listing, the UI should either prevent it or offer an "extend sponsorship" option.

## 6. Database Design

### 6.1. High-Level Schema:

*   **Users:** `user_id`, `email`, `password_hash`, `user_type`, `first_name`, `last_name`, `phone_number`, `profile_picture_url`, `created_at`, `updated_at`.
*   **Dealers:** `dealer_id` (FK to Users), `dealership_name`, `address`, `business_license_number`, `website_url`.
*   **Vehicles (Listings):** `vehicle_id`, `seller_id` (FK to Users), `make`, `model`, `year`, `vin`, `price`, `mileage`, `description`, `body_type`, `transmission_type`, `fuel_type`, `exterior_color`, `interior_color`, `engine_details`, `features` (JSON/separate table), `location_city`, `location_state_province`, `location_zip_code`, `status`, `created_at`, `updated_at`, `listed_date`, `sold_date`.
*   **Vehicle_Images:** `image_id`, `vehicle_id` (FK to Vehicles), `image_url`, `is_primary_image`, `uploaded_at`.
*   **Saved_Searches (Post-MVP):** `search_id`, `user_id` (FK to Users), `search_criteria` (JSON), `created_at`.
*   **Favorites (Post-MVP):** `favorite_id`, `user_id` (FK to Users), `vehicle_id` (FK to Vehicles), `created_at`.

### 6.2. Database Choice:

*   Relational database (e.g., PostgreSQL, MySQL).
*   Cloud-native options: Amazon RDS, Google Cloud SQL, Azure Database.

### 6.3. Database UML Class Diagram

This section contains a personal reference for the database schema.

This diagram focuses on the fully defined classes and relationships essential for the Minimum Viable Product (MVP). Post-MVP entities are also included for future context and are clearly marked.

```mermaid
classDiagram
    class User {
        +user_id: INTEGER (PK)
        +email: VARCHAR
        +password_hash: VARCHAR
        +user_type: VARCHAR (Enum: CUSTOMER, SELLER, DEALER, ADMIN)
        +first_name: VARCHAR
        +last_name: VARCHAR
        +phone_number: VARCHAR
        +profile_picture_url: VARCHAR
        +created_at: TIMESTAMP
        +updated_at: TIMESTAMP
        +getListings(): Vehicle[]
        +getFavorites(): Favorite[] (Post-MVP)
        +getSavedSearches(): SavedSearch[] (Post-MVP)
    }

    class Dealer {
        +dealer_id: INTEGER (PK, FK to User)
        +dealership_name: VARCHAR
        +address: VARCHAR
        +business_license_number: VARCHAR
        +website_url: VARCHAR
        +contact_email: VARCHAR
        +contact_phone: VARCHAR
        +subscription_tier: VARCHAR (Enum: BASIC, PREMIUM, PRO)
        +subscription_expiry_date: DATE
    }

    class Vehicle {
        +vehicle_id: INTEGER (PK)
        +seller_id: INTEGER (FK to User)
        +year: INTEGER
        +vin: VARCHAR
        +price: DECIMAL
        +mileage: INTEGER
        +description: TEXT
        +body_type: VARCHAR
        +transmission_type: VARCHAR
        +fuel_type: VARCHAR
        +exterior_color: VARCHAR
        +interior_color: VARCHAR
        +engine_details: VARCHAR
        +features: JSONB
        +location_city: VARCHAR
        +location_state_province: VARCHAR
        +location_zip_code: VARCHAR
        +status: VARCHAR (Enum: AVAILABLE, PENDING, SOLD, REMOVED)
        +created_at: TIMESTAMP
        +updated_at: TIMESTAMP
        +listed_date: DATE
        +sold_date: DATE
        +make_id: INTEGER (FK to VehicleMake)
        +model_id: INTEGER (FK to VehicleModel)
        +trim_id: INTEGER (FK to VehicleTrim)
        +getImages(): VehicleImage[]
        +getSeller(): User
    }

    class VehicleMake {
        +make_id: INTEGER (PK)
        +name: VARCHAR
    }

    class VehicleModel {
        +model_id: INTEGER (PK)
        +make_id: INTEGER (FK to VehicleMake)
        +name: VARCHAR
    }

    class VehicleTrim {
        +trim_id: INTEGER (PK)
        +model_id: INTEGER (FK to VehicleModel)
        +name: VARCHAR
        +year: INTEGER
    }

    class VehicleImage {
        +image_id: INTEGER (PK)
        +vehicle_id: INTEGER (FK to Vehicle)
        +image_url: VARCHAR
        +is_primary_image: BOOLEAN
        +uploaded_at: TIMESTAMP
    }

    User "1" -- "0..1" Dealer : is a (if user_type is DEALER)
    User "1 (Seller)" -- "0..*" Vehicle : lists / owns
    Vehicle "1" -- "0..*" VehicleImage : has
    VehicleMake "1" -- "0..*" VehicleModel : has
    VehicleModel "1" -- "0..*" VehicleTrim : has
    Vehicle "1" -- "1" VehicleTrim : has
    Vehicle "1" -- "1" VehicleModel : has
    Vehicle "1" -- "1" VehicleMake : has

    class UserReview {
        +review_id: INTEGER (PK)
        +reviewer_id: INTEGER (FK to User)
        +reviewed_seller_id: INTEGER (FK to User)
        +rating: INTEGER
        +comment: TEXT
        +created_at: TIMESTAMP
    }

    class Shortlist {
        +shortlist_id: INTEGER (PK)
        +user_id: INTEGER (FK to User)
        +name: VARCHAR
        +created_at: TIMESTAMP
        +updated_at: TIMESTAMP
    }

    class ShortlistVehicle {
        +shortlist_id: INTEGER (FK to Shortlist)
        +vehicle_id: INTEGER (FK to Vehicle)
        +added_at: TIMESTAMP
        +PRIMARY KEY (shortlist_id, vehicle_id)
    }

    class SavedSearch {
      +search_id: INTEGER (PK)
      +user_id: INTEGER (FK to User)
      +search_name: VARCHAR
      +search_criteria: JSONB
      +created_at: TIMESTAMP
      +last_run_at: TIMESTAMP
    }

    class VehicleAIAnalysis {
        +analysis_id: INTEGER (PK)
        +vehicle_id: INTEGER (FK to Vehicle, UNIQUE)
        +summary: TEXT
        +pros: JSONB
        +cons: JSONB
        +market_value_estimation: DECIMAL
        +sentiment_score: FLOAT
        +generated_at: TIMESTAMP
    }

    User "1 (Reviewer)" -- "0..*" UserReview : writes
    User "1 (Reviewed Seller)" -- "0..*" UserReview : receives review for
    User "1" -- "0..*" Shortlist : creates
    Shortlist "1" -- "0..*" ShortlistVehicle : contains
    Vehicle "1" -- "0..*" ShortlistVehicle : appears in
    User "1" -- "0..*" SavedSearch : creates
    Vehicle "1" -- "0..1" VehicleAIAnalysis : has analysis

    %% Post-MVP Entities (for context, can be added later)
    %% (Favorite has been replaced by Shortlist/ShortlistVehicle)
```

### 6.4. Local Development Setup (Dockerized PostgreSQL)

This section serves as a personal guide for setting up the local development environment.

This project uses a Dockerized PostgreSQL instance for local development.

**Prerequisites:**
*   Docker Desktop or Docker Engine installed and running.

**Steps to Set Up:**

1.  **Pull the PostgreSQL Docker Image (if you haven't already):**
    ```bash
    docker pull postgres
    ```

2.  **Run the PostgreSQL Container:**
    *   This command starts a PostgreSQL container named `autotrader-postgres-db`.
    *   It sets a password for the `postgres` superuser. **Replace `yourSuperStrongPassword123!` with your actual password.**
    *   It maps port `5432` on your host to port `5432` in the container.
    *   It creates a Docker volume named `autotrader_pgdata` for data persistence.
    ```bash
    docker run --name autotrader-postgres-db -e POSTGRES_PASSWORD=yourSuperStrongPassword123! -p 5432:5432 -v autotrader_pgdata:/var/lib/postgresql/data -d postgres
    ```
    *Wait a few seconds for the container to initialize.*

3.  **Create Application Database & User:**
    *   **Create Database (e.g., `autotrader_app_db`):**
        ```bash
        docker exec -it autotrader-postgres-db psql -U postgres -c "CREATE DATABASE autotrader_app_db;"
        ```
        *(You'll be prompted for the `POSTGRES_PASSWORD` set above).*
    *   **Create User (e.g., `autotrader_user`):**
        ```bash
        docker exec -it autotrader-postgres-db psql -U postgres -c "CREATE USER autotrader_user WITH PASSWORD 'anotherStrongPassword456?';"
        ```
        *(Replace `anotherStrongPassword456?` with your desired app user password. You'll be prompted for the `POSTGRES_PASSWORD`)*.
    *   **Grant Privileges:**
        ```bash
        docker exec -it autotrader-postgres-db psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE autotrader_app_db TO autotrader_user;"
        ```
        *(Prompted for `POSTGRES_PASSWORD`)*.
    *   **Grant Schema Privileges (Important for creating tables):**
        ```bash
        docker exec -it autotrader-postgres-db psql -U postgres -d autotrader_app_db -c "GRANT USAGE, CREATE ON SCHEMA public TO autotrader_user;"
        ```
        *(Prompted for `POSTGRES_PASSWORD`)*.

4.  **Apply the Schema (`schema.sql`):**
    *   **Copy schema file to container:**
        ```bash
        docker cp "schema.sql" autotrader-postgres-db:/tmp/schema.sql
        ```
        *(Ensure `schema.sql` is in your current directory or provide the full path).*
    *   **Execute schema file:**
        ```bash
        docker exec -it autotrader-postgres-db psql -U autotrader_user -d autotrader_app_db -f /tmp/schema.sql
        ```
        *(You'll be prompted for the `autotrader_user`'s password, e.g., `anotherStrongPassword456?`)*.

**Connection Details for Your Application:**
*   **Host:** `localhost`
*   **Port:** `5432`
*   **Database Name:** `autotrader_app_db`
*   **Username:** `autotrader_user`
*   **Password:** (The password you set for `autotrader_user`, e.g., `anotherStrongPassword456?`)
*   **Example Connection String:**
    `postgresql://autotrader_user:YOUR_APP_USER_PASSWORD@localhost:5432/autotrader_app_db`
    *(Replace `YOUR_APP_USER_PASSWORD` with the actual password for `autotrader_user`)*.

**Managing the Container:**
*   **Stop:** `docker stop autotrader-postgres-db`
*   **Start:** `docker start autotrader-postgres-db`
*   **View Logs:** `docker logs autotrader-postgres-db`
*   **Remove (deletes container but NOT the `autotrader_pgdata` volume):** `docker rm autotrader-postgres-db`
*   **Remove Volume (deletes all data!):** `docker volume rm autotrader_pgdata`

#### 6.4.1. How to Back Up and Restore a PostgreSQL Database using pgAdmin

This guide provides step-by-step instructions for personal reference on how to create a backup of your PostgreSQL database and how to restore it later using the pgAdmin graphical interface. This is a safe way to preserve your data before making significant changes, such as applying a new schema.

##### Step 1: Back Up Your Database

1.  **Open pgAdmin** and connect to your PostgreSQL server.
2.  In the browser tree on the left, find the database you want to back up (e.g., `postgres` or your main application database).
3.  **Right-click** on the database name and select **Backup...**.
4.  The "Backup" dialog will appear. Configure the following settings:
    *   **Filename**: Click the folder icon to choose a location and name for your backup file (e.g., `my_database_backup.dump`).
    *   **Format**: Select **Custom**. This format is generally the most flexible for restoring.
    *   **Encoding**: You can usually leave this as `UTF8`.
5.  You can review the other tabs for more advanced options, but the defaults are typically sufficient.
6.  Click the **Backup** button. A process will start, and you'll see a "Backup job created" message. Once it's finished, you'll get a success notification.

Your database is now safely backed up. You can proceed with your testing and schema changes.

##### Step 2: Restore Your Database (When You Want to Revert)

When you are ready to revert to the backed-up version, you will first need to clean up the modified database and then restore it from your backup file.

1.  **Drop the Modified Database**:
    *   In the pgAdmin browser tree, right-click the database you want to replace and select **Delete/Drop**. This will completely remove the database and all its contents.

2.  **Re-create an Empty Database**:
    *   Right-click on **Databases** and select **Create** > **Database...**.
    *   Give it the **exact same name** as the one you just dropped.
    *   Click **Save**.

3.  **Restore from Backup**:
    *   **Right-click** on your new, empty database and select **Restore...**.
    *   In the "Restore" dialog:
        *   Make sure the **Format** is set to **Custom or tar**.
        *   **Filename**: Click the folder icon and select the backup file you created earlier (e.g., `my_database_backup.dump`).
    *   Click the **Restore** button.

The restore process will run, and when complete, your database will be back in the exact state it was in when you created the backup.

### 6.5. SQL Server Local Development Setup

This section serves as a personal guide for setting up the local development environment with SQL Server.

For local development with SQL Server, especially for restoring `.bak` files:

*   **Connection String:** `Server=localhost\SQLEXPRESS;Database=master;Trusted_Connection=True;`
*   **Instance Name:** `SQLEXPRESS`
*   **Installation Media Folder:** `C:\SQL2022\Express_Enu`

## 7. API Design (RESTful Endpoints)

*   **Authentication (`/api/auth`):**
    *   `POST /api/auth/register` - Register a new user.
    *   `POST /api/auth/login` - Log in a user.

*   **Users (`/api/users`):**
    *   `GET /api/users/:id` - Get user data by ID.
    *   `PUT /api/users/:id` - Update user data by ID.
    *   `POST /api/users/:id/profile-picture` - Upload a profile picture for a user.
    *   `GET /api/users/:id/profile-picture` - Get a user's profile picture.

*   **Dealers (`/api/dealers`):**
    *   `POST /api/dealers/:id/banner-image` - Upload a banner image for a dealer.
    *   `GET /api/dealers/:id/banner` - Get a dealer's banner image.

*   **Vehicles (`/api/vehicles`):**
    *   `GET /api/vehicles` - Get all vehicle listings with pagination.
    *   `GET /api/vehicles/dealer/:id` - Get all vehicle listings by dealer ID.
    *   `POST /api/vehicles/create` - Create a new vehicle listing.
    *   `GET /api/vehicles/mine` - Get all vehicle listings for the authenticated user.
    *   `GET /api/vehicles/:id/shortlist-count` - Get the number of times a vehicle has been shortlisted.
    *   `GET /api/vehicles/:id` - Get a single vehicle listing by ID.
    *   `GET /api/vehicles/:vehicleId/images` - Get all images for a specific vehicle.
    *   `POST /api/vehicles/:vehicleId/images` - Upload images for a vehicle.
    *   `DELETE /api/vehicles/:vehicleId/images/:imageId` - Delete a specific vehicle image.
    *   `PUT /api/vehicles/:id` - Update a vehicle listing by ID.

*   **Images (`/api/images`):**
    *   `POST /api/images/presign` - Get a pre-signed URL for R2 image upload.
    *   `POST /api/images/complete` - Complete the image upload process (record in DB).
    *   `GET /api/images/raw/:vehicleId/:imageName` - Stream raw image from R2.
    *   `POST /api/images/delete-batch` - Delete multiple images from R2 and database.

*   **Shortlists (`/api/shortlists`):**
    *   `GET /api/shortlists` - Get all shortlists for the current user.
    *   `POST /api/shortlists` - Create a new shortlist.
    *   `GET /api/shortlists/vehicles` - Get all shortlisted vehicles for the current user.
    *   `POST /api/shortlists/vehicles` - Add or remove a vehicle from a shortlist.
    *   `GET /api/shortlists/:id` - Get a specific shortlist by ID with its vehicles.
    *   `DELETE /api/shortlists/:id` - Delete a shortlist.

*   **VPIC (Vehicle API Integration) (`/api`):**
    *   `GET /api/get-body-types` - Get body types for a given make, model, and year.
    *   `GET /api/get-body-classes` - Get body classes for a given make, model, and year.
    *   `GET /api/models` - Get models for a given make.

## 8. High-Level Architecture

```mermaid
graph TD
    User[User (Browser/Mobile)] -->|HTTPS| Frontend[React Frontend (Vite on Web Server/CDN)]
    Frontend -->|REST API (HTTPS)| Backend[Node.js Backend (Express.js on App Server/PaaS)]
    Backend -->|JDBC/ORM| Database[Relational Database (PostgreSQL/MySQL on DBaaS)]
    Backend -->|SDK| CloudStorage[Cloud Storage (Cloudflare R2)]

    subgraph "Client Tier"
        User
    end

    subgraph "Presentation Tier"
        Frontend
    end

    subgraph "Application Tier"
        Backend
    end

    subgraph "Data Tier"
        Database
        CloudStorage
    end

    AdminUser[Admin User] -->|HTTPS| AdminInterface[Admin Interface (Part of Frontend or Separate)]
    AdminInterface -->|REST API (HTTPS)| Backend
```

**Explanation:**

*   **User:** Interacts via browser/mobile.
*   **Frontend (React/Vite):** UI, presentation logic, API calls. Vite handles the development server and build process. Hosted on web server/CDN.
*   **Backend (Node.js/Express.js):** Business logic, API handling, DB interaction, cloud storage management. Hosted on app server/PaaS.
*   **Database (Relational):** Stores structured data. Managed DBaaS recommended.
*   **Cloud Storage:** Stores unstructured data (images/videos), specifically Cloudflare R2.
*   **Admin Interface:** Dedicated section or app for admin functions.
