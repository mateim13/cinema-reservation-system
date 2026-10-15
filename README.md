# Cinema Reservation System

An object-oriented C++17 application for comprehensive cinema reservation management, featuring a console interface and a REST server (CrowCpp) for a modern web frontend.

**Author:** Mandrescu Matei, group 3121b

---

## Key Features

### Domain & Data Model
- Entities: `Film`, `Sala`, `Proiectie`, `Rezervare`, `User`, `Angajat`, `Voucher`, `ProdusConsumabil`, `Bilet`
- **Design patterns:** Singleton (`Cinematograf`), Factory (`BiletFactory`), Polymorphism (`User → Angajat` hierarchy and ticket hierarchy)
- 10 custom exceptions (`Exceptii.h`) for validation (occupied seat, date in the past, failed authentication, etc.)

### Authentication & Roles (3 Roles)
- **Customer** — book tickets, view personal booking history, cancel reservations
- **Employee** — operational tasks: ticket validation, daily schedule, concession stand POS, reservations, snack inventory, VIP orders
- **Admin** — management tasks: movies, screenings, theaters, vouchers, users, statistics

### Reservations
- Multi-seat booking with a visual seating map (real-time occupied seats)
- Theater types: Standard / IMAX / VIP, with different layouts (central aisle vs. VIP seat pairs)
- **Row-based pricing** (tiers): front rows ×0.85, middle rows ×1.00, back rows ×1.20 (VIP theaters excluded)
- Vouchers validated by day and ticket type
- Loyalty card — automatic 10% discount
- Premium VIP extras (champagne, platters, etc.), ordered during booking and saved persistently
- **Ticket with a scannable QR code** (containing the reservation ID) + barcode in the booking confirmation

### Customer Account
- Booking history with **filters** (status, upcoming/past, movie search) and **sorting**
- Customers can cancel reservations directly (the seat becomes available again)
- Summary: active tickets + total amount spent

### Schedule & Search
- “Schedule” page with **search** by title and **filters** by genre, format (2D/3D), and day

### Employee Dashboard (Operations)
- **Ticket validation** by code (valid / canceled / nonexistent) — scan the QR code or enter the ID
- **Daily schedule** with occupancy rates for each screening
- **Concession stand POS** — snack sales at the counter (receipt including VAT)
- **Reservations** — ticket list with search/filter/sort, cancellation, and CSV export
- **Snack inventory** — restocking
- **Premium VIP orders**

### Admin Dashboard (Management)
- CRUD operations for movies and screenings (add / **edit** / **delete**, with safeguards preventing editing or deleting screenings with active reservations)
- **Occupancy rate** per screening (progress bar + percentage)
- Statistics: **revenue by movie** and **revenue by period** (week / month)
- Voucher management (add / activate / delete)
- Dynamically loaded theater table; **Customers** and **Employees** displayed separately
- CSV export of reservations (from the employee dashboard)

### Technical Features
- CSV file persistence — **incremental**: every change is saved immediately
- **Thread safety**: global mutex (`std::lock_guard`) on all state-changing routes → prevents double booking during concurrent requests
- REST server (CrowCpp) with CORS middleware

---

## Technologies

| Component | Technology |
|---|---|
| Language | C++17 |
| Build system | CMake ≥ 3.15 |
| REST server | CrowCpp |
| Networking | Asio |
| Dependencies | vcpkg (Windows) / Homebrew (macOS) |
| Frontend | HTML + CSS + vanilla JavaScript |
| Persistence | CSV files (relative paths) |

---

## Project Structure

```
├── include/                 # headers (domain classes + services)
│   ├── Film.h, Sala.h, Proiectie.h, Rezervare.h
│   ├── Bilet.h, BiletFactory.h
│   ├── User.h, Angajat.h
│   ├── Voucher.h, FormatAudio.h, ProdusConsumabil.h
│   ├── ApiSerializer.h      # header-only JSON serialization
│   ├── Cinematograf.h       # Singleton orchestrator
│   ├── CrowServer.h
│   ├── StorageService.h     # CSV persistence (relative paths!)
│   ├── ConsoleUI.h
│   └── Exceptii.h           # custom exceptions
├── src/
│   ├── main.cpp             # console entry point (CinemaApp)
│   ├── main_crow.cpp        # REST server entry point (CinemaServer)
│   ├── CrowServer.cpp       # all REST routes + CORS + locks
│   ├── ConsoleUI.cpp
│   ├── Cinematograf.cpp
│   └── [Film|Sala|Proiectie|Rezervare|Bilet|User|Angajat|Voucher|ProdusConsumabil|StorageService].cpp
├── web/
│   ├── index.html           # welcome + login (3 roles) + register + schedule + customer account
│   ├── style.css, app.js    # styles and logic for index.html
│   ├── client.html          # 4-step reservation flow (self-contained)
│   ├── admin.html           # admin dashboard (self-contained)
│   └── angajat.html         # employee operations dashboard (validation, schedule, POS, reservations, inventory, VIP)
├── data/                    # CSV files (persistent data)
│   ├── filme.csv, sali.csv, proiectii.csv
│   ├── rezervari.csv, useri.csv, preturi.csv
│   ├── vouchere.csv, snacks.csv, vip_extras.csv
├── CMakeLists.txt
└── README.md
```

---

## Building and Running

> **Important:** Always run the server from the **project root directory**, not from `build/`.
> `StorageService` uses relative paths (`data/*.csv`).

### macOS (Homebrew)

```bash
brew install cmake crow asio
cmake -B build -S . -DCMAKE_PREFIX_PATH="/opt/homebrew"
cmake --build build --target CinemaServer -j 4   # REST server
cmake --build build --target CinemaApp -j 4      # console application
./build/CinemaServer    # starts at http://localhost:8080
```

### Windows (MinGW + vcpkg)

```powershell
.\vcpkg\vcpkg install --triplet x64-windows
cmake -B build -S . `
  -DCMAKE_TOOLCHAIN_FILE="vcpkg/scripts/buildsystems/vcpkg.cmake" `
  -DCMAKE_PREFIX_PATH="vcpkg/installed/x64-windows" `
  -G "MinGW Makefiles"
cmake --build build --target CinemaServer -j 4
.\build\CinemaServer.exe
```

### Linux

```bash
sudo apt install cmake libboost-dev
cmake -B build -S .
cmake --build build -j 4
./build/CinemaServer
```

---

## Web Frontend

The application runs as **a single service**: `CinemaServer` (Crow) serves **both** the pages in `web/` **and** the API.
Start the server and open **`http://localhost:8080`** in your browser. No separate static file server is needed, and the API is available at `http://localhost:8080/api/`.

| Page | Role | Description |
|--------|-----|-----------|
| `index.html` | All | Welcome, login (Customer/Employee/Admin), registration, movie schedule, my account |
| `client.html` | Customer | Complete 4-step reservation flow |
| `angajat.html` | Employee | Operations dashboard: ticket validation (QR/code), schedule, concession stand POS, reservations, snack inventory, VIP orders |
| `admin.html` | Admin | Management: movies, theaters, screenings, statistics, vouchers, customers & employees |

### Customer Reservation Flow (`client.html`)
1. **Movie schedule** — grid with prices and genres
2. **Showtime selection** — screenings grouped by theater
3. **Seat selection** — visual map, occupied seats marked in real time, row-based pricing
4. **Snacks & Bar** — optional order + confirmation with complete details before payment

---

## REST Endpoints

### Public / Customer
| Method | Endpoint | Description |
|--------|----------|-----------|
| POST | `/api/login` | Authentication (`{username, parola, tip}`; `tip` = client/angajat/admin) |
| POST | `/api/register` | Customer account registration |
| GET | `/api/filme` | List of movies |
| GET | `/api/sali` | List of theaters |
| GET | `/api/proiectii/viitoare` | Upcoming screenings (embedded movie + theater) |
| GET | `/api/proiectii/vip` | Screenings in VIP theaters |
| GET | `/api/proiectii/<id>/locuri` | Occupied seats for a screening |
| GET | `/api/snacks` | Concession products with stock availability |
| POST | `/api/verifica-voucher` | Voucher validation + discount percentage |
| POST | `/api/rezerva` | Create a reservation |
| POST | `/api/comanda-snacks` | Pre-order snacks |
| POST | `/api/vip-extras` | Premium VIP extras (persisted) |
| GET | `/api/rezervarile-mele?username=` | A customer's reservations |
| POST | `/api/anuleaza` | Customer reservation cancellation |

### Employee
| Method | Endpoint | Description |
|--------|----------|-----------|
| GET | `/api/rezervare/<id>` | Ticket verification (valid / canceled / nonexistent) |

### Admin
| Method | Endpoint | Description |
|--------|----------|-----------|
| GET | `/api/admin/useri` | List of users |
| GET | `/api/admin/rezervari` | List of reservations (with `proiectieId`) |
| GET | `/api/admin/vouchere` | List of vouchers |
| GET | `/api/admin/vip-extras` | Premium VIP orders |
| POST | `/api/admin/anuleaza` | Cancel a reservation |
| POST | `/api/admin/card-fidelitate` | Toggle loyalty card |
| POST | `/api/admin/filme` | Add a movie |
| DELETE | `/api/admin/filme/<titlu>` | Delete a movie |
| POST | `/api/admin/proiectii` | Add a screening (prevents theater/time conflicts) |
| POST | `/api/admin/proiectii/edit` | Edit a screening (blocked if active reservations exist) |
| DELETE | `/api/admin/proiectii/<id>` | Delete a screening (blocked if active reservations exist) |
| POST | `/api/admin/vouchere/add` · `/toggle` · DELETE `/<cod>` | Manage vouchers |
| POST | `/api/admin/snacks/refill` | Restock inventory |

---

## Demo Credentials

| Role | Username | Password |
|-----|----------|--------|
| Admin | `admin` | `admin123` |
| Employee | `angajat1`, `angajat2` | `parola123` |
| Customer | `ana`, `andrei`, `ioana`, `vlad` | `parola123` |

Each demo customer has various reservations (upcoming and past, with different theaters and times).
The admin account is created automatically on the first run if it does not exist. New customers can register through the web interface.

---

## Theater Types and Pricing

| Type | Layout | Theater Multiplier | Tickets |
|-----|--------|--------------------|--------|
| Standard | Central aisle (e.g., 10×15) | ×1.00 | Adult, Student, Child, Senior |
| IMAX | Central aisle (e.g., 8×12) | ×1.50 | Adult, Student, Child, Senior |
| VIP | Pairs of 2 seats (e.g., 4×6) | ×2.00 | VIP, Adult |

**Final price formula:**
```
finalPrice = movieBasePrice × ticketMultiplier × theaterMultiplier × rowMultiplier
```

- **Row multiplier** (Standard/IMAX only): front (≤30%) ×0.85, middle (≤70%) ×1.00, back (>70%) ×1.20
- **Loyalty card:** additional 10% discount
- **Voucher:** percentage discount on the ticket price (validated by day and type)

---

## CSV Files

| File | Format |
|--------|--------|
| `filme.csv` | `titlu,durata,tip,gen,pretBaza` |
| `sali.csv` | `nume,tipSala,randuri,locuriPeRand` |
| `proiectii.csv` | `id,titluFilm,numeSala,dataOra,formatAudio` |
| `rezervari.csv` | `idRez,idProi,rand,loc,tipBilet,pret,username,anulat` |
| `useri.csv` | `username,parolaHash,rol,cardFidelitate` (Employee: `...,Angajat,program,sala`) |
| `preturi.csv` | `tipBilet,multiplicator` |
| `vouchere.csv` | `cod,reducere%,ziuaInt,tipuriExcluse\|,activ` |
| `snacks.csv` | `nume,descriere,pret,stoc` |
| `vip_extras.csv` | `idRezervare⇥extrasJSON` (TAB-delimited) |

Data is saved **incrementally** — every operation (reservation, cancellation, admin action) is immediately written to CSV.
Passwords are hashed (djb2) — sufficient for educational purposes, but not for production use.
