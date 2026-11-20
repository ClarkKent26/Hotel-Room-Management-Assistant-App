# Hotel-Room-Management-Assistant-App

"""
Hotel Room Management Assistant App
A console-based prototype written in Python only (standard library).

Menu:
  1.  Room Registration            (saved automatically)
  2.  Room Availability Checker    (shows Occupied automatically once booked)
  3.  Guest Booking                (check-in / check-out dates set automatically)
  4.  Check-in & Check-out
  5.  Check-in & Check-out Details
  6.  Facility Booking             (many bookings allowed, maximum 3 hours each)
  7.  Ratings Given by Guests After Their Stay
  8.  Hotel Summary & Report
  9.  Modify Room / Booking
  10. Cancel Room Booking          (asks for a reason, refund notice)
  11. Cancel Booking (Facility)
  12. Reset All Data               (deletes everything after confirmation)

All data is saved automatically to hotel_data.json (next to this file)
after every change and is loaded again when the program starts.
"""

import json
import os
from datetime import date, datetime, timedelta

# ---------------------------------------------------------------------------
# Settings and shared data
# ---------------------------------------------------------------------------
ROOM_TYPES = ("Single", "Double", "Suite")
ROOM_STATUSES = ("Available", "Occupied")
FACILITIES = ("Swimming Pool", "Gym", "Sauna Room", "BBQ Area",
              "Bar", "Game Room", "Event Hall")
MAX_FACILITY_HOURS = 3
OPEN_HOUR = 8     # facilities open at 08:00
CLOSE_HOUR = 22   # facilities close at 22:00
REFUND_MESSAGE = "YOUR MONEY WILL BE REFUNDED IN LESS THAN HALF AN HOUR,THANK YOU!!!!"

try:
    DATA_DIR = os.path.dirname(os.path.abspath(__file__))
except NameError:               # e.g. when pasted into an interactive shell
    DATA_DIR = os.getcwd()
DATA_FILE = os.path.join(DATA_DIR, "hotel_data.json")

# rooms:    room_no -> {"type", "rate", "status"}
# bookings: booking_id -> {"guest", "room", "nights", "cost", "status",
#                          "check_in_date", "check_out_date",
#                          "actual_check_in", "actual_check_out",
#                          "cancel_reason" (only when cancelled)}
#           status: Reserved -> Checked-in -> Completed, or Reserved -> Cancelled
# facility_bookings: facility_id -> {"booking_id", "guest", "room", "facility",
#                                    "date", "start", "end", "hours"}
# ratings:  booking_id -> {"guest", "room", "stars", "comment", "date"}
rooms = {}
bookings = {}
facility_bookings = {}
ratings = {}
counters = {"booking": 0, "facility": 0}

# Big block letters for the welcome banner (5 rows per letter)
BIG_FONT = {
    " ": ["   ", "   ", "   ", "   ", "   "],
    "H": ["#   #", "#   #", "#####", "#   #", "#   #"],
    "O": [" ### ", "#   #", "#   #", "#   #", " ### "],
    "T": ["#####", "  #  ", "  #  ", "  #  ", "  #  "],
    "E": ["#####", "#    ", "#### ", "#    ", "#####"],
    "L": ["#    ", "#    ", "#    ", "#    ", "#####"],
    "R": ["#### ", "#   #", "#### ", "#  # ", "#   #"],
    "M": ["#   #", "## ##", "# # #", "#   #", "#   #"],
    "A": [" ### ", "#   #", "#####", "#   #", "#   #"],
    "N": ["#   #", "##  #", "# # #", "#  ##", "#   #"],
    "G": [" ####", "#    ", "# ###", "#   #", " ### "],
}


# ---------------------------------------------------------------------------
# Big, easy-to-read display helpers
# ---------------------------------------------------------------------------
def show_banner():
    """Print the welcome title in big block letters."""
    print()
    for word in ("HOTEL ROOM", "MANAGEMENT"):
        for row in range(5):
            line = ""
            for ch in word:
                line += BIG_FONT.get(ch, BIG_FONT[" "])[row] + " "
            print(line.rstrip())
        print()


def heading(text):
    """Print a large, boxed, upper-case section heading."""
    line = "=" * 64
    print()
    print(line)
    print(text.upper().center(64))
    print(line)


def notice(text):
    """Print an important message inside a big star box."""
    width = len(text) + 8
    print()
    print("*" * width)
    print("*" + " " * (width - 2) + "*")
    print("*" + text.center(width - 2) + "*")
    print("*" + " " * (width - 2) + "*")
    print("*" * width)
    print()


# ---------------------------------------------------------------------------
# Automatic saving and loading
# ---------------------------------------------------------------------------
def save_data():
    """Write all data to the JSON file. Called after every change."""
    data = {
        "rooms": rooms,
        "bookings": bookings,
        "facility_bookings": facility_bookings,
        "ratings": ratings,
        "counters": counters,
    }
    try:
        temp_file = DATA_FILE + ".tmp"
        with open(temp_file, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2)
        os.replace(temp_file, DATA_FILE)
    except OSError as error:
        print(f"WARNING: could not save data ({error})")


def load_data():
    """Load saved data (if any) when the program starts."""
    if not os.path.exists(DATA_FILE):
        return
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
        rooms.update(data.get("rooms", {}))
        bookings.update(data.get("bookings", {}))
        facility_bookings.update(data.get("facility_bookings", {}))
        ratings.update(data.get("ratings", {}))
        counters.update(data.get("counters", {}))
        print(f"SAVED DATA LOADED: {len(rooms)} room(s), {len(bookings)} booking(s).")
    except (OSError, ValueError, AttributeError):
        print("WARNING: the saved data file could not be read. Starting empty.")
        try:
            os.replace(DATA_FILE, DATA_FILE + ".corrupt")
        except OSError:
            pass


# ---------------------------------------------------------------------------
# Small helpers
# ---------------------------------------------------------------------------
def next_id(kind, prefix):
    """Generate IDs such as B001 (booking) or F001 (facility)."""
    counters[kind] += 1
    return f"{prefix}{counters[kind]:03d}"


def today_str():
    return date.today().isoformat()


def now_str():
    return datetime.now().strftime("%Y-%m-%d %H:%M")


def active_booking_for_room(room_no):
    """Return (booking_id, booking) of the Reserved/Checked-in booking, or (None, None)."""
    for booking_id, b in bookings.items():
        if b["room"] == room_no and b["status"] in ("Reserved", "Checked-in"):
            return booking_id, b
    return None, None


def has_active_reservation(room_no):
    return active_booking_for_room(room_no)[0] is not None


def to_minutes(hhmm):
    hours, minutes = hhmm.split(":")
    return int(hours) * 60 + int(minutes)


def to_hhmm(minutes):
    return f"{minutes // 60:02d}:{minutes % 60:02d}"


# ---------------------------------------------------------------------------
# Input validation helpers
# ---------------------------------------------------------------------------
def read_text(prompt):
    while True:
        value = input(prompt).strip()
        if value:
            return value
        print("Input cannot be empty!")


def read_positive_int(prompt):
    while True:
        try:
            value = int(input(prompt))
            if value > 0:
                return value
        except ValueError:
            pass
        print("Only positive integer is allowed!")


def read_positive_float(prompt):
    while True:
        try:
            value = float(input(prompt))
            if value > 0:
                return value
        except ValueError:
            pass
        print("Only positive number is allowed!")


def read_choice(title, options):
    print(title)
    for i, option in enumerate(options, start=1):
        print(f"  {i}. {option}")
    while True:
        try:
            number = int(input(f"Enter your choice (1-{len(options)}): "))
            if 1 <= number <= len(options):
                return options[number - 1]
        except ValueError:
            pass
        print("Invalid choice!")


def read_yes_no(prompt):
    while True:
        answer = input(prompt + " (y/n): ").strip().lower()
        if answer in ("y", "yes"):
            return True
        if answer in ("n", "no"):
            return False
        print("Please enter y or n!")


def read_stars():
    while True:
        try:
            stars = int(input("Enter rating (1 to 5 stars): "))
            if 1 <= stars <= 5:
                return stars
        except ValueError:
            pass
        print("Only a whole number from 1 to 5 is allowed!")


def read_hours():
    while True:
        try:
            hours = int(input(f"Enter number of hours (1 to {MAX_FACILITY_HOURS}): "))
            if 1 <= hours <= MAX_FACILITY_HOURS:
                return hours
            if hours > MAX_FACILITY_HOURS:
                print(f"Maximum limit is {MAX_FACILITY_HOURS} hours per booking!")
            else:
                print("Minimum booking is 1 hour!")
        except ValueError:
            print("Only whole numbers are allowed!")


def read_date(low, high):
    """Read a date between low and high (ISO strings). Enter = low."""
    while True:
        text = input(f"Enter date (YYYY-MM-DD, {low} to {high}, Enter = {low}): ").strip()
        if text == "":
            return low
        try:
            chosen = datetime.strptime(text, "%Y-%m-%d").date().isoformat()
        except ValueError:
            print("Use the format YYYY-MM-DD!")
            continue
        if low <= chosen <= high:
            return chosen
        print(f"The date must be between {low} and {high}!")


# ---------------------------------------------------------------------------
# Table display helpers
# ---------------------------------------------------------------------------
def show_rooms(room_numbers):
    if not room_numbers:
        print("No rooms found.")
        return
    print(f"{'Room':<8}{'Type':<10}{'Rate (RM/night)':<17}{'Status':<11}{'Guest':<14}{'Until':<12}")
    print("-" * 72)
    for number in room_numbers:
        room = rooms[number]
        _, b = active_booking_for_room(number)
        guest = b["guest"][:12] if b else "-"
        until = b["check_out_date"] if b else "-"
        print(f"{number:<8}{room['type']:<10}{room['rate']:<17.2f}"
              f"{room['status']:<11}{guest:<14}{until:<12}")


def show_bookings(booking_ids):
    print(f"{'ID':<6}{'Guest':<16}{'Room':<8}{'Nights':<8}{'Cost (RM)':<12}{'Status':<12}")
    print("-" * 62)
    for booking_id in booking_ids:
        b = bookings[booking_id]
        print(f"{booking_id:<6}{b['guest'][:15]:<16}{b['room']:<8}"
              f"{b['nights']:<8}{b['cost']:<12.2f}{b['status']:<12}")


def show_facility_bookings(facility_ids):
    print(f"{'ID':<6}{'Facility':<15}{'Guest':<13}{'Room':<7}{'Date':<12}{'Time':<13}{'Hours':<6}")
    print("-" * 72)
    for fid in facility_ids:
        f = facility_bookings[fid]
        time_range = f["start"] + "-" + f["end"]
        print(f"{fid:<6}{f['facility']:<15}{f['guest'][:12]:<13}{f['room']:<7}"
              f"{f['date']:<12}{time_range:<13}{f['hours']:<6}")


# ---------------------------------------------------------------------------
# 1. Room Registration
# ---------------------------------------------------------------------------
def register_room():
    heading("Room Registration")
    room_no = read_text("Enter room number: ").upper()
    if room_no in rooms:
        print(f"Room {room_no} is already registered!")
        return

    room_type = read_choice("Select room type:", ROOM_TYPES)
    rate = read_positive_float("Enter nightly rate (RM): ")
    status = read_choice("Select availability status:", ROOM_STATUSES)

    rooms[room_no] = {"type": room_type, "rate": rate, "status": status}
    save_data()
    print(f"\nROOM {room_no} HAS BEEN REGISTERED AND SAVED AUTOMATICALLY.")
    print()
    show_rooms(sorted(rooms))


# ---------------------------------------------------------------------------
# 2. Room Availability Checker
# ---------------------------------------------------------------------------
def check_availability():
    heading("Room Availability Checker")
    if not rooms:
        print("No rooms registered yet.")
        return

    option = read_choice(
        "Filter rooms by:",
        ("All rooms", "Availability status", "Room type"),
    )

    if option == "All rooms":
        matches = sorted(rooms)
    elif option == "Availability status":
        status = read_choice("Select status:", ROOM_STATUSES)
        matches = sorted(n for n in rooms if rooms[n]["status"] == status)
    else:
        room_type = read_choice("Select room type:", ROOM_TYPES)
        matches = sorted(n for n in rooms if rooms[n]["type"] == room_type)

    print()
    show_rooms(matches)


# ---------------------------------------------------------------------------
# 3. Guest Booking
# ---------------------------------------------------------------------------
def book_room():
    heading("Guest Booking")

    bookable = sorted(
        n for n in rooms
        if rooms[n]["status"] == "Available" and not has_active_reservation(n)
    )
    if not bookable:
        print("No rooms are available for booking.")
        return

    print("AVAILABLE ROOMS:")
    show_rooms(bookable)
    print()

    guest = read_text("Enter guest name: ")

    while True:
        room_no = read_text("Enter room number to book: ").upper()
        if room_no in bookable:
            break
        print("Please select a room from the available list!")

    nights = read_positive_int("Enter number of nights: ")

    booking_id = next_id("booking", "B")
    check_in = date.today()
    check_out = check_in + timedelta(days=nights)
    cost = rooms[room_no]["rate"] * nights

    bookings[booking_id] = {
        "guest": guest,
        "room": room_no,
        "nights": nights,
        "cost": cost,
        "status": "Reserved",
        "check_in_date": check_in.isoformat(),
        "check_out_date": check_out.isoformat(),
        "actual_check_in": None,
        "actual_check_out": None,
    }
    rooms[room_no]["status"] = "Occupied"      # automatic update
    save_data()

    print("\nBOOKING CREATED SUCCESSFULLY!")
    print(f"Booking ID      : {booking_id}")
    print(f"Guest           : {guest}")
    print(f"Room            : {room_no} ({rooms[room_no]['type']})")
    print(f"Nights          : {nights}")
    print(f"Check-in date   : {check_in.isoformat()}")
    print(f"Check-out date  : {check_out.isoformat()}")
    print(f"Estimated cost  : RM {cost:.2f}")
    print(f"Room {room_no} is now marked OCCUPIED.")


# ---------------------------------------------------------------------------
# 4. Check-in & Check-out
# ---------------------------------------------------------------------------
def check_in():
    heading("Check-in")
    booking_id = read_text("Enter Booking ID: ").upper()

    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return
    booking = bookings[booking_id]

    if booking["status"] == "Checked-in":
        print("This guest has already checked in!")
        return
    if booking["status"] == "Completed":
        print("This booking has already been completed!")
        return
    if booking["status"] == "Cancelled":
        print("This booking has been cancelled!")
        return

    booking["status"] = "Checked-in"
    booking["actual_check_in"] = now_str()
    rooms[booking["room"]]["status"] = "Occupied"
    save_data()
    print(f"\n{booking['guest']} HAS CHECKED IN TO ROOM {booking['room']}.")
    print(f"Check-in time   : {booking['actual_check_in']}")
    print(f"Check-out due   : {booking['check_out_date']}")


def check_out():
    heading("Check-out")
    booking_id = read_text("Enter Booking ID: ").upper()

    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return
    booking = bookings[booking_id]

    if booking["status"] == "Reserved":
        print("This guest has not checked in yet!")
        return
    if booking["status"] == "Completed":
        print("This booking has already been completed!")
        return
    if booking["status"] == "Cancelled":
        print("This booking has been cancelled!")
        return

    booking["status"] = "Completed"
    booking["actual_check_out"] = now_str()
    rooms[booking["room"]]["status"] = "Available"   # automatic update

    # Facility bookings for later days are no longer needed
    removed = [fid for fid, f in facility_bookings.items()
               if f["booking_id"] == booking_id and f["date"] > today_str()]
    for fid in removed:
        del facility_bookings[fid]
    save_data()

    print(f"\n{booking['guest']} HAS CHECKED OUT OF ROOM {booking['room']}.")
    print(f"Check-out time  : {booking['actual_check_out']}")
    print(f"Total amount due: RM {booking['cost']:.2f}")
    print(f"Room {booking['room']} is now AVAILABLE again.")
    if removed:
        print(f"{len(removed)} future facility booking(s) were cancelled automatically.")
    print("Tip: use option 7 to record the guest's rating for this stay.")


def checkin_checkout_menu():
    option = read_choice("Check-in & Check-out:", ("Check-in", "Check-out"))
    if option == "Check-in":
        check_in()
    else:
        check_out()


# ---------------------------------------------------------------------------
# 5. Check-in & Check-out Details
# ---------------------------------------------------------------------------
def stay_details():
    heading("Check-in & Check-out Details")
    if not bookings:
        print("No bookings yet.")
        return

    print(f"{'ID':<6}{'Guest':<13}{'Room':<7}{'Check-in':<12}{'Check-out':<12}"
          f"{'Actual in':<18}{'Actual out':<18}{'Status':<11}")
    print("-" * 97)
    for booking_id in sorted(bookings):
        b = bookings[booking_id]
        actual_in = b["actual_check_in"] or "-"
        actual_out = b["actual_check_out"] or "-"
        print(f"{booking_id:<6}{b['guest'][:12]:<13}{b['room']:<7}"
              f"{b['check_in_date']:<12}{b['check_out_date']:<12}"
              f"{actual_in:<18}{actual_out:<18}{b['status']:<11}")

    cancelled = [b for b in sorted(bookings) if bookings[b]["status"] == "Cancelled"]
    if cancelled:
        print("\nCANCELLATION REASONS:")
        for booking_id in cancelled:
            reason = bookings[booking_id].get("cancel_reason", "-")
            print(f"  {booking_id}: {reason}")


# ---------------------------------------------------------------------------
# 6. Facility Booking
# ---------------------------------------------------------------------------
def show_booked_times(facility, day):
    """Show only the booked TIMES for a facility on a date (no guest names)."""
    slots = sorted((f["start"], f["end"]) for f in facility_bookings.values()
                   if f["facility"] == facility and f["date"] == day)
    print()
    print(f"BOOKED TIMES FOR {facility.upper()} ON {day}")
    print(f"(open {OPEN_HOUR:02d}:00 to {CLOSE_HOUR:02d}:00)")
    if not slots:
        print("   No bookings yet - the whole day is free.")
    else:
        for start, end in slots:
            print(f"   {start} to {end}   - already booked")
    print()


def book_one_facility(booking_id, booking):
    """Book one facility. Returns True when the booking was made."""
    facility = read_choice("Select a facility:", FACILITIES)

    day = read_date(today_str(), booking["check_out_date"])
    show_booked_times(facility, day)
    hours = read_hours()

    while True:
        text = input(f"Enter start time (HH:MM, {OPEN_HOUR:02d}:00 to {CLOSE_HOUR:02d}:00): ").strip()
        try:
            parsed = datetime.strptime(text, "%H:%M")
        except ValueError:
            print("Use the 24-hour format HH:MM, for example 14:30!")
            continue
        start_min = parsed.hour * 60 + parsed.minute
        end_min = start_min + hours * 60

        if start_min < OPEN_HOUR * 60 or end_min > CLOSE_HOUR * 60:
            print(f"Facilities are open {OPEN_HOUR:02d}:00 to {CLOSE_HOUR:02d}:00. "
                  "Your booking must fit inside these hours!")
            continue

        clash = False
        for f in facility_bookings.values():
            if f["date"] != day:
                continue
            overlaps = start_min < to_minutes(f["end"]) and end_min > to_minutes(f["start"])
            if not overlaps:
                continue
            if f["facility"] == facility:
                print(f"{facility} is already booked {f['start']}-{f['end']}. "
                      "Please choose another time!")
                clash = True
                break
            if f["booking_id"] == booking_id:
                print(f"You already have {f['facility']} booked {f['start']}-{f['end']} "
                      "at that time!")
                clash = True
                break
        if clash:
            continue
        break

    facility_id = next_id("facility", "F")
    facility_bookings[facility_id] = {
        "booking_id": booking_id,
        "guest": booking["guest"],
        "room": booking["room"],
        "facility": facility,
        "date": day,
        "start": to_hhmm(start_min),
        "end": to_hhmm(end_min),
        "hours": hours,
    }
    save_data()

    print("\nFACILITY BOOKED SUCCESSFULLY!")
    print(f"Facility ID : {facility_id}")
    print(f"Guest       : {booking['guest']} (room {booking['room']})")
    print(f"Facility    : {facility}")
    print(f"Date        : {day}")
    print(f"Time        : {to_hhmm(start_min)} to {to_hhmm(end_min)} ({hours} hour(s))")
    return True


def book_facility():
    heading("Facility Booking")
    in_house = sorted(b for b in bookings if bookings[b]["status"] == "Checked-in")
    if not in_house:
        print("No guests are currently checked in.")
        print("Facilities can only be booked by checked-in guests.")
        return

    print("CHECKED-IN GUESTS:")
    show_bookings(in_house)
    print()
    booking_id = read_text("Enter your Booking ID: ").upper()
    if booking_id not in in_house:
        print("Invalid Booking ID, or the guest is not checked in!")
        return
    booking = bookings[booking_id]

    if today_str() > booking["check_out_date"]:
        print("This guest's booked stay has ended. Please check out first.")
        return

    # Multiple entries: keep booking facilities until the guest says no
    while True:
        book_one_facility(booking_id, booking)
        print()
        if not read_yes_no("Do you want to book another facility?"):
            break
        print()

    mine = sorted((fid for fid, f in facility_bookings.items() if f["booking_id"] == booking_id),
                  key=lambda k: (facility_bookings[k]["date"], facility_bookings[k]["start"]))
    print("\nYOUR FACILITY BOOKINGS:")
    show_facility_bookings(mine)


def view_facility_bookings():
    heading("Facility Bookings")
    if not facility_bookings:
        print("No facility bookings yet.")
        return
    ordered = sorted(facility_bookings,
                     key=lambda k: (facility_bookings[k]["date"], facility_bookings[k]["start"]))
    show_facility_bookings(ordered)


def facility_menu():
    option = read_choice("Facility Booking:", ("Book a facility", "View facility bookings"))
    if option == "Book a facility":
        book_facility()
    else:
        view_facility_bookings()


# ---------------------------------------------------------------------------
# 7. Ratings Given by Guests After Their Stay
# ---------------------------------------------------------------------------
def give_rating():
    heading("Give a Rating")
    waiting = sorted(b for b in bookings
                     if bookings[b]["status"] == "Completed" and b not in ratings)
    if not waiting:
        print("No completed stays are waiting for a rating.")
        return

    print("COMPLETED STAYS WITHOUT A RATING:")
    show_bookings(waiting)
    print()
    booking_id = read_text("Enter Booking ID to rate: ").upper()
    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return
    booking = bookings[booking_id]
    if booking["status"] != "Completed":
        print("Ratings can only be given after the guest has checked out!")
        return
    if booking_id in ratings:
        print("This stay has already been rated!")
        return

    stars = read_stars()
    comment = input("Enter a comment (optional, press Enter to skip): ").strip()

    ratings[booking_id] = {
        "guest": booking["guest"],
        "room": booking["room"],
        "stars": stars,
        "comment": comment,
        "date": today_str(),
    }
    save_data()
    print(f"\nTHANK YOU, {booking['guest'].upper()}! YOUR {stars}/5 RATING HAS BEEN SAVED.")


def view_ratings():
    heading("Guest Ratings")
    if not ratings:
        print("No ratings have been given yet.")
        return

    print(f"{'ID':<6}{'Guest':<14}{'Room':<7}{'Rating':<8}{'Date':<12}Comment")
    print("-" * 70)
    total = 0
    for booking_id in sorted(ratings):
        r = ratings[booking_id]
        total += r["stars"]
        stars_text = f"{r['stars']}/5"
        print(f"{booking_id:<6}{r['guest'][:13]:<14}{r['room']:<7}"
              f"{stars_text:<8}{r['date']:<12}{r['comment'] or '-'}")
    print(f"\nAVERAGE RATING: {total / len(ratings):.1f}/5 from {len(ratings)} guest(s)")


def ratings_menu():
    option = read_choice("Ratings Given by Guests After Their Stay:",
                         ("Give a rating", "View all ratings"))
    if option == "Give a rating":
        give_rating()
    else:
        view_ratings()


# ---------------------------------------------------------------------------
# 8. Hotel Summary & Report
# ---------------------------------------------------------------------------
def hotel_summary():
    heading("Hotel Summary & Report")
    total_rooms = len(rooms)
    occupied = sum(1 for r in rooms.values() if r["status"] == "Occupied")

    completed = 0
    cancelled = 0
    revenue = 0.0
    for booking in bookings.values():
        if booking["status"] == "Cancelled":
            cancelled += 1
            continue
        revenue += booking["cost"]
        if booking["status"] == "Completed":
            completed += 1

    occupancy = occupied / total_rooms * 100 if total_rooms > 0 else 0.0
    facility_hours = sum(f["hours"] for f in facility_bookings.values())

    print(f"Total registered rooms          : {total_rooms}")
    print(f"Occupied rooms                  : {occupied}")
    print(f"Occupancy rate                  : {occupancy:.1f}%")
    print(f"Total bookings                  : {len(bookings)}")
    print(f"Completed stays                 : {completed}")
    print(f"Cancelled bookings              : {cancelled}")
    print(f"Estimated accommodation revenue : RM {revenue:.2f}")
    print(f"Facility bookings               : {len(facility_bookings)} ({facility_hours} hour(s) in total)")
    if ratings:
        average = sum(r["stars"] for r in ratings.values()) / len(ratings)
        print(f"Average guest rating            : {average:.1f}/5 ({len(ratings)} rating(s))")
    else:
        print("Average guest rating            : no ratings yet")

    if bookings:
        print("\nBOOKING RECORDS:")
        show_bookings(sorted(bookings))


# ---------------------------------------------------------------------------
# 9. Modify Room / Booking
# ---------------------------------------------------------------------------
def modify_room():
    heading("Modify Room")
    if not rooms:
        print("No rooms registered yet.")
        return

    show_rooms(sorted(rooms))
    print()
    room_no = read_text("Enter room number to modify: ").upper()
    if room_no not in rooms:
        print("Room not found!")
        return
    room = rooms[room_no]

    field = read_choice("What do you want to modify?",
                        ("Room type", "Nightly rate", "Availability status"))

    if field == "Room type":
        room["type"] = read_choice("Select new room type:", ROOM_TYPES)
    elif field == "Nightly rate":
        room["rate"] = read_positive_float("Enter new nightly rate (RM): ")
        print("Note: existing bookings keep their original cost.")
    else:
        if has_active_reservation(room_no):
            print("Cannot change status: the room has an active booking!")
            return
        room["status"] = read_choice("Select new status:", ROOM_STATUSES)

    save_data()
    print(f"\nROOM {room_no} HAS BEEN UPDATED AND SAVED.")
    show_rooms([room_no])


def modify_booking():
    heading("Modify Booking")
    reserved = sorted(b for b in bookings if bookings[b]["status"] == "Reserved")
    if not reserved:
        print("There are no reserved bookings to modify.")
        return

    print("RESERVED BOOKINGS:")
    show_bookings(reserved)
    print()
    booking_id = read_text("Enter Booking ID to modify: ").upper()
    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return
    booking = bookings[booking_id]
    if booking["status"] != "Reserved":
        print("Only reserved bookings can be modified!")
        return

    field = read_choice("What do you want to modify?", ("Guest name", "Number of nights"))

    if field == "Guest name":
        booking["guest"] = read_text("Enter new guest name: ")
    else:
        rate_per_night = booking["cost"] / booking["nights"]
        booking["nights"] = read_positive_int("Enter new number of nights: ")
        booking["cost"] = rate_per_night * booking["nights"]
        start = date.fromisoformat(booking["check_in_date"])
        booking["check_out_date"] = (start + timedelta(days=booking["nights"])).isoformat()

    save_data()
    print(f"\nBOOKING {booking_id} HAS BEEN UPDATED AND SAVED.")
    show_bookings([booking_id])
    print(f"Check-out date: {booking['check_out_date']}")


def modify_menu():
    option = read_choice("Modify:", ("Modify room", "Modify booking"))
    if option == "Modify room":
        modify_room()
    else:
        modify_booking()


# ---------------------------------------------------------------------------
# 10. Cancel Room Booking
# ---------------------------------------------------------------------------
def cancel_booking():
    heading("Cancel Room Booking")
    reserved = sorted(b for b in bookings if bookings[b]["status"] == "Reserved")
    if not reserved:
        print("There are no reserved bookings to cancel.")
        return

    print("RESERVED BOOKINGS:")
    show_bookings(reserved)
    print()
    booking_id = read_text("Enter Booking ID to cancel: ").upper()
    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return
    booking = bookings[booking_id]

    if booking["status"] == "Checked-in":
        print("This guest has already checked in. Please check out instead!")
        return
    if booking["status"] == "Completed":
        print("This booking has already been completed!")
        return
    if booking["status"] == "Cancelled":
        print("This booking has already been cancelled!")
        return

    if not read_yes_no(f"Cancel booking {booking_id} for {booking['guest']}?"):
        print("Cancellation aborted. No changes made.")
        return

    reason = read_text("What is the reason for cancelling? ")

    booking["status"] = "Cancelled"
    booking["cancel_reason"] = reason
    rooms[booking["room"]]["status"] = "Available"   # automatic update
    save_data()

    print(f"\nBOOKING {booking_id} HAS BEEN CANCELLED.")
    print(f"Reason : {reason}")
    print(f"Room {booking['room']} is now AVAILABLE again.")
    notice(REFUND_MESSAGE)


# ---------------------------------------------------------------------------
# 11. Cancel Booking (Facility)
# ---------------------------------------------------------------------------
def cancel_facility_booking():
    heading("Cancel Booking (Facility)")
    if not facility_bookings:
        print("There are no facility bookings to cancel.")
        return

    booking_id = read_text("Enter your Booking ID: ").upper()
    if booking_id not in bookings:
        print("Invalid Booking ID!")
        return

    mine = sorted((fid for fid, f in facility_bookings.items() if f["booking_id"] == booking_id),
                  key=lambda k: (facility_bookings[k]["date"], facility_bookings[k]["start"]))
    if not mine:
        print("This guest has no facility bookings.")
        return

    print("\nYOUR FACILITY BOOKINGS:")
    show_facility_bookings(mine)
    print()
    facility_id = read_text("Enter Facility ID to cancel: ").upper()
    if facility_id not in mine:
        print("Invalid Facility ID for this guest!")
        return
    f = facility_bookings[facility_id]

    question = f"Cancel {f['facility']} on {f['date']} ({f['start']}-{f['end']})?"
    if read_yes_no(question):
        del facility_bookings[facility_id]
        save_data()
        print(f"\nFACILITY BOOKING {facility_id} HAS BEEN CANCELLED.")
        print("The time slot is now free for other guests.")
    else:
        print("Cancellation aborted. No changes made.")


# ---------------------------------------------------------------------------
# 12. Reset All Data
# ---------------------------------------------------------------------------
def reset_all_data():
    heading("Reset All Data")
    print("This will permanently delete:")
    print(f"   - {len(rooms)} room(s)")
    print(f"   - {len(bookings)} room booking(s)")
    print(f"   - {len(facility_bookings)} facility booking(s)")
    print(f"   - {len(ratings)} rating(s)")
    notice("WARNING: THIS CANNOT BE UNDONE!")

    if not read_yes_no("Are you sure you want to delete ALL data?"):
        print("Reset aborted. No data was deleted.")
        return

    rooms.clear()
    bookings.clear()
    facility_bookings.clear()
    ratings.clear()
    counters["booking"] = 0
    counters["facility"] = 0
    save_data()
    print("\nALL DATA HAS BEEN DELETED. THE SYSTEM IS NOW EMPTY.")
    print("Booking IDs will start again from B001.")


# ---------------------------------------------------------------------------
# Main menu
# ---------------------------------------------------------------------------
def main():
    show_banner()
    load_data()
    while True:
        heading("Hotel Room Management Assistant")
        print("   1.  ROOM REGISTRATION")
        print("   2.  ROOM AVAILABILITY CHECKER")
        print("   3.  GUEST BOOKING")
        print("   4.  CHECK-IN & CHECK-OUT")
        print("   5.  CHECK-IN & CHECK-OUT DETAILS")
        print("   6.  FACILITY BOOKING")
        print("   7.  RATINGS GIVEN BY GUESTS AFTER THEIR STAY")
        print("   8.  HOTEL SUMMARY & REPORT")
        print("   9.  MODIFY ROOM / BOOKING")
        print("   10. CANCEL ROOM BOOKING")
        print("   11. CANCEL BOOKING (FACILITY)")
        print("   12. RESET ALL DATA")
        print("   0.  EXIT")
        print()
        option = input("ENTER AN OPTION (0-12): ").strip()

        if option == "1":
            register_room()
        elif option == "2":
            check_availability()
        elif option == "3":
            book_room()
        elif option == "4":
            checkin_checkout_menu()
        elif option == "5":
            stay_details()
        elif option == "6":
            facility_menu()
        elif option == "7":
            ratings_menu()
        elif option == "8":
            hotel_summary()
        elif option == "9":
            modify_menu()
        elif option == "10":
            cancel_booking()
        elif option == "11":
            cancel_facility_booking()
        elif option == "12":
            reset_all_data()
        elif option == "0":
            print("\nTHANK YOU! SEE YOU AGAIN!")
            break
        else:
            print("WRONG INPUT! Please enter a number from 0 to 12.")


if __name__ == "__main__":
    main()
