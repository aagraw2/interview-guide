
## 1. Functional Requirements

- User can search movies by city
    
- View theaters and show timings
    
- View seat layout for a show
    
- Select seats and create booking
    
- Make payment and confirm booking
    
- Prevent double booking of seats
    
- Cancel booking (optional basic support)
    

---

## 2. Non-Functional Requirements

- High concurrency (multiple users booking same show)
    
- Low latency seat availability checks
    
- Strong consistency for seat booking (no double booking)
    
- Scalable across cities, theaters, and shows
    
- Fault-tolerant payment handling
    
- Extensible for offers, seat types, pricing
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|User|Represents customer|
|Movie|Movie details|
|City|Groups theaters|
|Theater|Contains screens|
|Screen|Contains seats|
|Show|Movie screening instance|
|Seat|Represents seat in screen|
|ShowSeat|Seat availability for a specific show|
|Booking|Stores booking details|
|Payment|Handles payment info|
|PricingStrategy|Calculates ticket price|

---

## 4. Flows (Use Cases)

### Search & Browse (Main Flow)

1. User selects city
    
2. System lists movies running in city
    
3. User selects movie
    
4. System shows theaters and show timings
    

---

### Seat Selection (Main Flow)

1. User selects a show
    
2. System fetches ShowSeat layout
    
3. User selects seats
    
4. System temporarily locks seats
    

---

### Booking (Main Flow)

1. User proceeds to book selected seats
    
2. Validate seat availability
    
3. Create booking with status "PENDING"
    
4. Initiate payment
    

---

### Payment & Confirmation (Main Flow)

1. User completes payment
    
2. If success:
    
    - Mark booking as CONFIRMED
        
    - Mark seats as BOOKED
        
3. If failure:
    
    - Release locked seats
        
    - Mark booking FAILED
        

---

### Seat Locking (Supporting Flow)

1. On seat selection → mark seats as LOCKED
    
2. Associate lock with user/session
    
3. Start timeout (e.g., 5 mins)
    
4. If timeout expires → release seats
    

---

## 5. Code

```java
import java.util.*;

// MODELS

enum SeatStatus {
    AVAILABLE, LOCKED, BOOKED
}

enum BookingStatus {
    PENDING, CONFIRMED, FAILED
}

class User {
    String id;
}

class Movie {
    String id;
    String name;
}

class Seat {
    int id;
    int row;
}

class ShowSeat {
    Seat seat;
    SeatStatus status;

    public ShowSeat(Seat seat) {
        this.seat = seat;
        this.status = SeatStatus.AVAILABLE;
    }
}

class Show {
    String id;
    Movie movie;
    List<ShowSeat> seats;

    public Show(String id, Movie movie, List<ShowSeat> seats) {
        this.id = id;
        this.movie = movie;
        this.seats = seats;
    }
}

class Booking {
    String id;
    User user;
    Show show;
    List<ShowSeat> seats;
    BookingStatus status;

    public Booking(String id, User user, Show show, List<ShowSeat> seats) {
        this.id = id;
        this.user = user;
        this.show = show;
        this.seats = seats;
        this.status = BookingStatus.PENDING;
    }
}

// Strategy Pattern for pricing
interface PricingStrategy {
    double calculatePrice(List<ShowSeat> seats);
}

class DefaultPricingStrategy implements PricingStrategy {
    @Override
    public double calculatePrice(List<ShowSeat> seats) {
        return seats.size() * 200.0;
    }
}

// SERVICES

// Singleton Booking Service
class BookingService {
    private static BookingService instance;

    private Map<String, Booking> bookings = new HashMap<>();
    private PricingStrategy pricingStrategy;

    private BookingService() {
        this.pricingStrategy = new DefaultPricingStrategy();
    }

    public static BookingService getInstance() {
        if (instance == null) {
            synchronized (BookingService.class) {
                if (instance == null) {
                    instance = new BookingService();
                }
            }
        }
        return instance;
    }

    // Lock seats
    public synchronized boolean lockSeats(List<ShowSeat> seats) {
        for (ShowSeat seat : seats) {
            if (seat.status != SeatStatus.AVAILABLE) return false;
        }
        for (ShowSeat seat : seats) {
            seat.status = SeatStatus.LOCKED;
        }
        return true;
    }

    // Create booking
    public Booking createBooking(User user, Show show, List<ShowSeat> seats) {
        if (!lockSeats(seats)) {
            throw new RuntimeException("Seats not available");
        }

        Booking booking = new Booking(UUID.randomUUID().toString(), user, show, seats);
        bookings.put(booking.id, booking);

        return booking;
    }

    // Confirm booking after payment
    public synchronized void confirmBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);

        for (ShowSeat seat : booking.seats) {
            seat.status = SeatStatus.BOOKED;
        }

        booking.status = BookingStatus.CONFIRMED;
    }

    // Fail booking
    public synchronized void failBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);

        for (ShowSeat seat : booking.seats) {
            seat.status = SeatStatus.AVAILABLE;
        }

        booking.status = BookingStatus.FAILED;
    }

    public double calculateAmount(String bookingId) {
        Booking booking = bookings.get(bookingId);
        return pricingStrategy.calculatePrice(booking.seats);
    }
}
```

---

## 6. Complexity Analysis

### Seat Locking

- Validate seats → O(K)
    
- Update status → O(K)
    

**Overall: O(K)**

---

### Booking Creation

- Lock seats → O(K)
    
- Insert booking → O(1)
    

**Overall: O(K)**

---

### Confirm Booking

- Update seats → O(K)
    

**Overall: O(K)**

---

### Payment Calculation

- Iterate seats → O(K)
    

**Overall: O(K)**

---

## 7. Key Design Decisions

- Separate ShowSeat from Seat to handle per-show availability
    
- Synchronized methods ensure no double booking (basic thread safety)
    
- Strategy Pattern allows flexible pricing rules
    
- Singleton BookingService centralizes booking logic
    
- Seat locking prevents race conditions during booking
    
- In-memory storage used; can be replaced with DB + distributed locks for scale