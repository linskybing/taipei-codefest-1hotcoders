# Taipei Codefest - 1hotcoders Project Technical Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Frontend Implementation](#frontend-implementation)
   - [Map System](#map-system)
   - [Bidirectional Communication](#bidirectional-communication)
   - [Quest Management](#quest-management)
4. [Backend Implementation](#backend-implementation)
   - [Data Models](#data-models)
   - [API Endpoints](#api-endpoints)
   - [Data Processing Flow](#data-processing-flow)
5. [Key Features](#key-features)
6. [Deployment and Environment](#deployment-and-environment)

---

## Project Overview

This is an interactive application that combines **fitness tracking** with **virtual pet raising**. Users nurture their virtual pets through actual exercise and can explore Taipei city landmarks to complete check-in quests on an interactive map.

### Technology Stack

**Frontend**
- React 18 + TypeScript
- Vite (Build tool)
- React Router v6 (Routing)
- Leaflet + react-leaflet (Map implementation)
- Tailwind CSS + Shadcn/ui (UI framework)
- Sonner (Notification system)

**Backend**
- FastAPI (Python web framework)
- SQLAlchemy (ORM)
- PostgreSQL (Database)
- Pydantic (Data validation)

**Deployment**
- Frontend: Vercel
- Backend: Railway
- Database: Railway PostgreSQL

---

## System Architecture

```mermaid
graph TB
    subgraph Frontend["Frontend (React + TypeScript)"]
        A[User Interface] --> B[Router]
        B --> C[Map Component]
        B --> D[Exercise Page]
        B --> E[Status Page]
        C --> F[Leaflet Map]
        D --> G[GPS Location]
        E --> H[Quest List]
    end
    
    subgraph API["API Communication Layer"]
        I[api.ts]
        J[HTTP Client]
    end
    
    subgraph Backend["Backend (FastAPI)"]
        K[FastAPI Routes]
        L[CRUD Business Logic]
        M[Data Validation]
    end
    
    subgraph Database["Database (PostgreSQL)"]
        N[Users]
        O[Pets]
        P[Exercise Logs]
        Q[Travel Checkins]
    end
    
    Frontend --> API
    API --> Backend
    Backend --> Database
    
    style Frontend fill:#e1f5ff
    style Backend fill:#fff4e1
    style Database fill:#e8f5e9
```

---

## Frontend Implementation

### Map System

#### 1. Core Technologies

The interactive map functionality is implemented using **Leaflet** and **react-leaflet**.

```typescript
// Map container configuration
<MapContainer
  center={[userLocation.lat, userLocation.lng]}
  zoom={12}
  style={{ height: '100%', width: '100%' }}
  zoomControl={true}
  minZoom={11}
  maxZoom={18}
>
  <TileLayer
    url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
    attribution='&copy; OpenStreetMap contributors'
  />
  {/* Other components */}
</MapContainer>
```

**Key Features:**
- Uses OpenStreetMap as the base map
- Supports standard operations like zoom and pan
- Responsive design adaptable to various screen sizes

#### 2. Real-time GPS Positioning

Implements real-time location tracking using the **Geolocation API**:

```typescript
// hooks/useLocation.ts
const { getLocation } = useLocation();

useEffect(() => {
  const fetchLocation = async () => {
    const locationData = await getLocation();
    if (locationData && locationData.success) {
      setUserLocation({
        lat: locationData.latitude,
        lng: locationData.longitude,
        accuracy: locationData.accuracy,
      });
      setHasRealLocation(true);
    }
  };

  fetchLocation();
  
  // Update location every 5 seconds
  const interval = setInterval(fetchLocation, 5000);
  
  return () => clearInterval(interval);
}, []);
```

**Location Flow:**
1. Request location permission on initial load
2. Obtain initial position
3. Update position every 5 seconds
4. Display location accuracy range (blue circle)

#### 3. Quest Markers and Visualization

Display different colored markers based on quest status:

```typescript
const createQuestIcon = (status: QuestLocation['status'], category: string) => {
  const color = status === 'completed' ? '#22c55e' :   // Green - Completed
                status === 'in-progress' ? '#f59e0b' : // Orange - In Progress
                category === '運動場館' ? '#ec4899' :   // Pink - Sports Venue
                '#8b5cf6';                             // Purple - General Location

  return new L.Icon({
    iconUrl: 'data:image/svg+xml;base64,' + btoa(`
      <svg xmlns="http://www.w3.org/2000/svg" width="40" height="40" viewBox="0 0 24 24">
        <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z" fill="${color}"/>
        <circle cx="12" cy="10" r="3" fill="white" stroke="${color}"/>
      </svg>
    `),
    iconSize: [40, 40],
    iconAnchor: [20, 40],
    popupAnchor: [0, -40],
  });
};
```

**Quest Status Descriptions:**
- **Available**: Purple marker, can be accepted within 1000 meters
- **In Progress**: Orange marker, can be completed within 100 meters
- **Completed**: Green marker, cannot be completed again

#### 4. Geofence Validation

Uses the **Haversine formula** to calculate distance between two points:

```typescript
const calculateDistance = (lat1: number, lng1: number, lat2: number, lng2: number): number => {
  const R = 6371e3; // Earth radius in meters
  const φ1 = (lat1 * Math.PI) / 180;
  const φ2 = (lat2 * Math.PI) / 180;
  const Δφ = ((lat2 - lat1) * Math.PI) / 180;
  const Δλ = ((lng2 - lng1) * Math.PI) / 180;

  const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
            Math.cos(φ1) * Math.cos(φ2) * Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

  return R * c;
};

const isInRange = (quest: QuestLocation): boolean => {
  if (devMode) return true; // Always in range in developer mode
  const distance = calculateDistance(
    userLocation.lat, userLocation.lng,
    quest.lat, quest.lng
  );
  const requiredDistance = quest.status === 'available' ? 1000 : 100;
  return distance <= requiredDistance;
};
```

#### 5. Route Navigation

Uses **Leaflet Routing Machine** to provide route planning:

```typescript
const RoutingControl = ({ start, end }: { start: [number, number]; end: [number, number] }) => {
  const map = useMap();
  
  useEffect(() => {
    const routingControl = L.Routing.control({
      waypoints: [
        L.latLng(start[0], start[1]),
        L.latLng(end[0], end[1])
      ],
      routeWhileDragging: false,
      lineOptions: {
        styles: [{ color: '#6366f1', weight: 4, opacity: 0.7 }]
      },
      createMarker: () => null,
      router: L.Routing.osrmv1({
        serviceUrl: 'https://router.project-osrm.org/route/v1'
      })
    }).addTo(map);

    return () => {
      map.removeControl(routingControl);
    };
  }, [map, start, end]);

  return null;
};
```

---

### Bidirectional Communication

#### 1. API Communication Architecture

All API requests are handled through a unified `api.ts` module:

```typescript
// lib/api.ts
const API_BASE_URL = "https://back-end-production-68f7.up.railway.app";

// Create user
export async function createUser(pet_name: string, townpass_id?: string): Promise<User> {
  const response = await fetch(`${API_BASE_URL}/users/`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ pet_name, user_id: townpass_id }),
  });
  if (!response.ok) throw new Error("Failed to create user");
  return response.json();
}

// Log exercise
export async function logExercise(userId: string, log: ExerciseLogCreate): Promise<ExerciseResult> {
  const response = await fetch(`${API_BASE_URL}/users/${userId}/exercise`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(log),
  });
  if (!response.ok) throw new Error("Failed to log exercise");
  return response.json();
}

// Create travel check-in
export async function createTravelCheckin(
  userId: string, 
  checkin: TravelCheckinCreate
): Promise<{ pet: Pet; checkin: TravelCheckin }> {
  const response = await fetch(`${API_BASE_URL}/users/${userId}/travel/checkins`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(checkin),
  });
  if (!response.ok) throw new Error("Failed to create travel checkin");
  return response.json();
}
```

#### 2. State Management

Uses **React Context** to manage global user state:

```typescript
// hooks/useUser.tsx
const UserContext = createContext<UserContextType | undefined>(undefined);

export function UserProvider({ children }: { children: React.ReactNode }) {
  const [userId, setUserId] = useState<string | null>(null);
  const [pet, setPet] = useState<Pet | null>(null);
  const [loading, setLoading] = useState(true);

  const refreshPet = useCallback(async () => {
    if (!userId) return;
    try {
      const petData = await getUserPet(userId);
      setPet(petData);
    } catch (error) {
      console.error("Failed to refresh pet:", error);
    }
  }, [userId]);

  useEffect(() => {
    const storedUserId = localStorage.getItem("userId");
    if (storedUserId) {
      setUserId(storedUserId);
      refreshPet();
    }
    setLoading(false);
  }, []);

  return (
    <UserContext.Provider value={{ userId, pet, setUserId, setPet, refreshPet, loading }}>
      {children}
    </UserContext.Provider>
  );
}
```

#### 3. Data Flow

**Exercise Logging Flow:**

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant A as API Layer
    participant B as Backend
    participant D as Database

    U->>F: Start exercise
    F->>F: Start timer
    U->>F: End exercise
    F->>A: logExercise(userId, data)
    A->>B: POST /users/{id}/exercise
    B->>D: Save exercise log
    B->>D: Update pet status
    D-->>B: Return updated Pet
    B-->>A: {pet, breakthrough_required}
    A-->>F: Update local state
    F->>F: refreshPet()
    F-->>U: Show reward notification
```

**Check-in Flow:**

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant G as GPS
    participant A as API Layer
    participant B as Backend
    participant D as Database

    U->>F: Click map marker
    F->>G: Get current location
    G-->>F: {lat, lng, accuracy}
    F->>F: Check distance
    alt Within range
        F->>A: createTravelCheckin(userId, data)
        A->>B: POST /users/{id}/travel/checkins
        B->>D: Check for duplicate
        alt Not duplicate
            B->>D: Save checkin record
            B->>D: Update pet rewards
            D-->>B: Return update result
            B-->>A: {pet, checkin}
            A-->>F: Update state
            F-->>U: Show success notification
        else Already checked in
            B-->>A: Error: Already checked in
            A-->>F: Error message
            F-->>U: Show error prompt
        end
    else Out of range
        F-->>U: Show distance prompt
    end
```

#### 4. Error Handling

Unified error handling mechanism:

```typescript
// Using try-catch in components
const handleCompleteQuest = async (quest: QuestLocation) => {
  if (!userId) {
    toast.error("Please log in first");
    return;
  }

  try {
    const result = await createTravelCheckin(userId, {
      quest_id: quest.id,
      lat: quest.lat,
      lng: quest.lng
    });

    // Success handling
    setQuests(prev => prev.map(q =>
      q.id === quest.id ? { ...q, status: "completed" } : q
    ));
    
    toast.success("Check-in successful!", {
      description: `Rewards earned: ${bonusText}`
    });
  } catch (error) {
    console.error("Failed to complete quest:", error);
    toast.error("Check-in failed, please try again later");
  }
};
```

#### 5. Data Persistence

Uses **localStorage** to store user ID:

```typescript
// Save user ID
localStorage.setItem("userId", userId);

// Read user ID
const storedUserId = localStorage.getItem("userId");

// Clear user ID (logout)
localStorage.removeItem("userId");
```

---

### Quest Management

#### 1. Quest Data Structure

```typescript
interface QuestLocation {
  id: string;
  name: string;
  description: string;
  bonus: {
    strength?: number;
    mood?: number;
  };
  category: "景點" | "公園" | "運動場館";
  lat: number;
  lng: number;
  status: "available" | "in-progress" | "completed";
}
```

#### 2. Quest State Machine

```mermaid
stateDiagram-v2
    [*] --> available: Quest initialized
    available --> in_progress: User accepts quest<br/>(within 1km range)
    in_progress --> completed: User completes check-in<br/>(within 100m range)
    in_progress --> available: User cancels quest
    completed --> [*]: Quest ended
    
    note right of available
        Purple marker
        1000m acceptance range
    end note
    
    note right of in_progress
        Orange marker
        100m completion range
        Shows navigation route
    end note
    
    note right of completed
        Green marker
        Cannot repeat
    end note
```

#### 3. Dual Mode Toggle

**Map Mode**: Intuitively view quest locations
**List Mode**: Browse quests by category

```typescript
const [viewMode, setViewMode] = useState<"map" | "list">("map");

// Mode toggle logic
{viewMode === "map" ? (
  <MapView
    quests={quests}
    onAcceptQuest={handleAcceptQuest}
    onCompleteQuest={handleCompleteQuest}
    devMode={devMode}
    flyToQuest={flyToQuest}
    onFlyComplete={() => setFlyToQuest(null)}
    activeQuestId={activeQuestId}
  />
) : (
  <QuestList
    availableQuests={availableQuests}
    inProgressQuests={inProgressQuests}
    completedQuests={completedQuests}
    onViewOnMap={handleViewOnMap}
  />
)}
```

---

## Backend Implementation

### Data Models

#### 1. User

```python
class User(Base):
    __tablename__ = "users"
    
    id = Column(String, primary_key=True, index=True)  # TownPass ID
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    # Relationships
    pet = relationship("Pet", back_populates="owner", uselist=False)
    exercise_logs = relationship("ExerciseLog", back_populates="user")
    quests = relationship("UserQuest", back_populates="user")
    travel_checkins = relationship("TravelCheckin", back_populates="user")
```

#### 2. Pet

```python
class Pet(Base):
    __tablename__ = "pets"
    
    id = Column(Integer, primary_key=True)
    name = Column(String, default="我的手雞")
    owner_id = Column(String, ForeignKey("users.id"))
    
    # Pet attributes
    strength = Column(Integer, default=0)           # Strength (0-120)
    stamina = Column(Integer, default=100)          # Stamina (0-900)
    mood = Column(Integer, default=0)               # Mood (0-100)
    level = Column(Integer, default=1)              # Level
    stage = Column(Enum(PetStage), default=PetStage.EGG)  # Growth stage
    
    # Daily tracking
    daily_exercise_seconds = Column(Integer, default=0)    # Today's exercise seconds
    daily_steps = Column(Integer, default=0)               # Today's steps
    last_reset_date = Column(DateTime(timezone=True))      # Last reset time
    
    # Breakthrough system
    breakthrough_completed = Column(Boolean, default=False)
    last_daily_check = Column(DateTime(timezone=True))
    
    # Daily quest status
    daily_quest_1_completed = Column(Boolean, default=False)
    daily_quest_2_completed = Column(Boolean, default=True)
    daily_quest_3_completed = Column(Boolean, default=True)
```

**Growth Stage Definitions:**

```python
class PetStage(int, enum.Enum):
    EGG = 0          # Egg (Level 1-4)
    CHICK = 1        # Chick (Level 5-9, requires lv5 breakthrough)
    CHICKEN = 2      # Chicken (Level 10-14, requires lv10 breakthrough)
    BIG_CHICKEN = 3  # Big Chicken (Level 15-19, requires lv15 breakthrough)
    BUFF_CHICKEN = 4 # Buff Chicken (Level 20+, requires lv20 breakthrough)
```

#### 3. ExerciseLog

```python
class ExerciseLog(Base):
    __tablename__ = "exercise_logs"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(String, ForeignKey("users.id"))
    pet_id = Column(Integer, ForeignKey("pets.id"))
    
    exercise_type = Column(String)       # "Stationary", "Running", "Walking"
    duration_seconds = Column(Integer)   # Exercise duration (seconds)
    volume = Column(Float)               # Exercise intensity coefficient
    steps = Column(Integer, default=0)   # Steps (for walking/running)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

#### 4. TravelCheckin

```python
class TravelCheckin(Base):
    __tablename__ = "travel_checkins"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(String, ForeignKey("users.id"))
    quest_id = Column(String, index=True)  # Corresponds to frontend quest ID
    completed_at = Column(DateTime(timezone=True), server_default=func.now())
    lat = Column(Float)  # Check-in latitude
    lng = Column(Float)  # Check-in longitude
```

---

### API Endpoints

#### 1. User Management

```python
# Create user
@app.post("/users/", response_model=schemas.User)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    """
    Create new user and automatically create pet
    - pet_name: Pet name (required)
    - user_id: TownPass ID (optional)
    """
    return crud.create_user(db=db, user=user)

# Get user information
@app.get("/users/{user_id}", response_model=schemas.User)
def read_user(user_id: str, db: Session = Depends(get_db)):
    """Get user information (includes pet status)"""
    db_user = crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user
```

#### 2. Pet Management

```python
# Get pet status
@app.get("/users/{user_id}/pet", response_model=schemas.Pet)
def get_user_pet(user_id: str, db: Session = Depends(get_db)):
    """Get user's pet current status"""
    pet = crud.get_pet_by_user_id(db, user_id=user_id)
    if pet is None:
        raise HTTPException(status_code=404, detail="Pet not found")
    return pet

# Update pet attributes
@app.patch("/users/{user_id}/pet", response_model=schemas.Pet)
def update_user_pet(user_id: str, pet_update: schemas.PetUpdate, db: Session = Depends(get_db)):
    """
    Update pet attributes
    Can update: name, strength, stamina, mood, level, stage, breakthrough_completed
    """
    pet = crud.get_pet_by_user_id(db, user_id)
    if not pet:
        raise HTTPException(status_code=404, detail="Pet not found")
    return crud.update_pet(db, pet, pet_update)
```

#### 3. Exercise Logging

```python
@app.post("/users/{user_id}/exercise")
def log_exercise(user_id: str, log: schemas.ExerciseLogCreate, db: Session = Depends(get_db)):
    """
    Log exercise
    
    Parameters:
    - exercise_type: Exercise type (Stationary, Running, Walking)
    - duration_seconds: Exercise duration (seconds)
    - steps: Step count (for walking/running)
    
    Returns:
    - pet: Updated pet status
    - breakthrough_required: Whether breakthrough is needed
    
    Calculation rules:
    - 10 seconds exercise = +1 strength point
    - 10 seconds exercise = -1 stamina point
    - Each exercise = +5 mood
    - 120 strength points = level up
    """
    result = crud.log_exercise(db, user_id, log)
    if result is None:
        raise HTTPException(status_code=404, detail="User or pet not found")
    return result
```

#### 4. Travel Check-in

```python
# Create check-in record
@app.post("/users/{user_id}/travel/checkins")
def create_travel_checkin(
    user_id: str, 
    checkin: schemas.TravelCheckinCreate, 
    db: Session = Depends(get_db)
):
    """
    Check in at a location
    
    Parameters:
    - quest_id: Quest ID
    - lat, lng: Check-in location
    
    Features:
    - Check for duplicate check-ins
    - Record check-in location and time
    - Give pet rewards (strength +20, mood +10)
    """
    try:
        result = crud.create_travel_checkin(db, user_id, checkin)
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

# Get check-in records
@app.get("/users/{user_id}/travel/checkins", response_model=List[schemas.TravelCheckin])
def get_user_travel_checkins(user_id: str, db: Session = Depends(get_db)):
    """Get all user check-in records"""
    return crud.get_user_travel_checkins(db, user_id)
```

#### 5. Daily Check

```python
@app.post("/users/{user_id}/daily-check")
def perform_daily_check(user_id: str, db: Session = Depends(get_db)):
    """
    Perform daily check
    
    Check items:
    1. Is it a new day
    2. Did user exercise enough yesterday (at least 10 minutes)
    3. Reset daily statistics (stamina, steps, exercise time)
    
    Penalty mechanism:
    - If didn't exercise yesterday and stamina > 0: mood -10
    - If mood = 0 and strength > 0: strength -10
    """
    result = crud.perform_daily_check(db, user_id)
    if result is None:
        raise HTTPException(status_code=404, detail="User or pet not found")
    return result
```

#### 6. Breakthrough System

```python
# Complete breakthrough
@app.post("/users/{user_id}/travel/breakthrough")
def complete_breakthrough(user_id: str, db: Session = Depends(get_db)):
    """
    Complete breakthrough quest
    
    Trigger condition: Pet reaches level 5, 10, 15, or 20
    Effects:
    - Remove strength gain restriction
    - Evolve to next stage (EGG → CHICK → CHICKEN → BIG_CHICKEN → BUFF_CHICKEN)
    """
    result = crud.complete_breakthrough(db, user_id)
    if result is None:
        raise HTTPException(status_code=404, detail="Pet not found")
    return result

# Start breakthrough quest (get random attraction)
@app.post("/users/{user_id}/travel/start", response_model=schemas.Attraction)
def start_travel_quest(user_id: str, db: Session = Depends(get_db)):
    """Get random Taipei attraction as breakthrough quest target"""
    pet = crud.get_pet_by_user_id(db, user_id)
    if not pet:
        raise HTTPException(status_code=404, detail="Pet not found")
    
    if pet.level % 5 != 0 or pet.level < 5:
        raise HTTPException(status_code=400, detail="Not at breakthrough level")
    
    if pet.breakthrough_completed:
        raise HTTPException(status_code=400, detail="Breakthrough already completed")
    
    attraction = crud.get_random_attraction(db)
    return attraction
```

---

### Data Processing Flow

#### 1. Exercise Logging Processing

```python
def log_exercise(db: Session, user_id: str, log: schemas.ExerciseLogCreate):
    """Process exercise log and update pet status"""
    pet = get_pet_by_user_id(db, user_id)
    if not pet:
        return None
    
    # 1. Calculate strength gain (10 seconds = 1 point)
    strength_gain = log.duration_seconds // 10
    
    # 2. Calculate stamina cost (10 seconds = 1 point)
    stamina_cost = -(log.duration_seconds // 10)
    
    # 3. Mood increase (+5 per session)
    mood_gain = 5
    
    # 4. Accumulate daily statistics
    pet.daily_exercise_seconds += log.duration_seconds
    pet.daily_steps += log.steps
    
    # 5. Update pet status (includes level-up logic)
    result = update_pet_stats(
        db=db, 
        pet=pet,
        strength=strength_gain,
        stamina=stamina_cost,
        mood=mood_gain
    )
    
    # 6. Save exercise log
    exercise_log = models.ExerciseLog(
        user_id=user_id,
        pet_id=pet.id,
        exercise_type=log.exercise_type,
        duration_seconds=log.duration_seconds,
        volume=log.volume,
        steps=log.steps
    )
    db.add(exercise_log)
    db.commit()
    
    return result
```

#### 2. Pet Level-up System

```python
def update_pet_stats(db: Session, pet: models.Pet, 
                     strength: int = 0, stamina: int = 0, mood: int = 0):
    """Update pet attributes and handle level-ups"""
    
    # 1. Check if breakthrough is needed
    needs_breakthrough = (
        pet.level % 5 == 0 and 
        pet.level >= 5 and 
        not pet.breakthrough_completed
    )
    
    if needs_breakthrough and strength > 0:
        # Block strength gain, return breakthrough prompt
        pet.stamina = max(0, min(MAX_STAMINA, pet.stamina + stamina))
        pet.mood = max(0, min(100, pet.mood + mood))
        db.commit()
        return {"pet": pet, "breakthrough_required": True}
    
    # 2. Apply strength gain
    pet.strength += strength
    
    # 3. Handle level-up (120 strength = 1 level)
    while pet.strength >= STRENGTH_PER_LEVEL and pet.level < MAX_LEVEL:
        pet.strength -= STRENGTH_PER_LEVEL
        pet.level += 1
        
        # Level-up rewards
        pet.stamina = MAX_STAMINA  # Full stamina
        pet.mood += 10             # Mood +10
        
        # Check if reached breakthrough level
        if pet.level % 5 == 0 and pet.level >= 5:
            pet.breakthrough_completed = False
            break  # Stop leveling, wait for breakthrough
    
    # 4. Update stamina and mood
    pet.stamina = max(0, min(MAX_STAMINA, pet.stamina + stamina))
    pet.mood = max(0, min(100, pet.mood + mood))
    
    # 5. Update growth stage
    pet.stage = get_stage_for_level(pet.level, pet.breakthrough_completed)
    
    db.commit()
    db.refresh(pet)
    
    return {
        "pet": pet, 
        "breakthrough_required": False
    }
```

#### 3. Daily Check Logic

```python
def perform_daily_check(db: Session, user_id: str):
    """Execute daily check and reset"""
    pet = get_pet_by_user_id(db, user_id)
    if not pet:
        return None
    
    now = datetime.now(timezone.utc)
    last_check = pet.last_reset_date
    
    # Determine if it's a new day
    is_new_day = (
        last_check is None or 
        last_check.date() < now.date()
    )
    
    if not is_new_day:
        return {
            "pet": pet,
            "already_checked": True,
            "met_requirement": True
        }
    
    # Check if yesterday's exercise goal was met (60 strength points = 10 minutes)
    met_requirement = pet.daily_exercise_seconds >= MIN_DAILY_STRENGTH * 10
    
    # If didn't meet goal and stamina > 0, deduct mood
    if not met_requirement and pet.stamina > 0:
        pet.mood = max(0, pet.mood - 10)
        
        # If mood reaches zero and strength > 0, deduct strength
        if pet.mood == 0 and pet.strength > 0:
            pet.strength = max(0, pet.strength - 10)
    
    # Reset daily statistics
    pet.stamina = MAX_STAMINA
    pet.daily_exercise_seconds = 0
    pet.daily_steps = 0
    pet.last_reset_date = now
    
    # Reset daily quests
    pet.daily_quest_1_completed = False
    pet.daily_quest_2_completed = True
    pet.daily_quest_3_completed = True
    
    db.commit()
    db.refresh(pet)
    
    return {
        "pet": pet,
        "already_checked": False,
        "met_requirement": met_requirement
    }
```

#### 4. Check-in Processing Flow

```python
def create_travel_checkin(db: Session, user_id: str, checkin_data: schemas.TravelCheckinCreate):
    """Process location check-in"""
    pet = get_pet_by_user_id(db, user_id)
    if not pet:
        raise ValueError("Pet not found")
    
    # 1. Check for duplicate check-in
    existing = db.query(models.TravelCheckin).filter(
        models.TravelCheckin.user_id == user_id,
        models.TravelCheckin.quest_id == checkin_data.quest_id
    ).first()
    
    if existing:
        raise ValueError("Already checked in at this location")
    
    # 2. Create check-in record
    checkin = models.TravelCheckin(
        user_id=user_id,
        quest_id=checkin_data.quest_id,
        lat=checkin_data.lat,
        lng=checkin_data.lng
    )
    db.add(checkin)
    
    # 3. Give rewards
    reward_result = update_pet_stats(
        db=db,
        pet=pet,
        strength=20,  # Strength +20
        mood=10       # Mood +10
    )
    
    db.commit()
    db.refresh(checkin)
    
    return {
        "pet": reward_result["pet"],
        "checkin": checkin
    }
```

---

## Key Features

### 1. Game Mechanics Design

#### Strength System
- **How to Gain**: 10 seconds of exercise = +1 strength point
- **Level-up Condition**: Accumulate 120 strength points = level up (requires 20 minutes of exercise)
- **Reset Mechanism**: Strength resets to zero after leveling up

#### Stamina System
- **Daily Quota**: 900 stamina points (can exercise for 150 minutes)
- **Consumption Rate**: 10 seconds of exercise = -1 stamina point
- **Reset Time**: Resets at 00:00 or on first login each day

#### Mood System
- **How to Increase**: +5 per exercise session, +10 on level-up
- **Penalty Mechanism**: If didn't exercise enough previous day (<10 minutes) and stamina > 0, mood -10
- **Severe Warning**: When mood reaches zero, starts deducting strength (-10/day)

#### Breakthrough System
- **Trigger Levels**: 5, 10, 15, 20
- **Restriction Effect**: Cannot gain strength until breakthrough is completed
- **How to Break Through**: Check in at designated Taipei landmark
- **Evolution Reward**: Pet evolves to next stage upon completion

### 2. Growth Stages

```
Level 1-4    →  Egg (EGG)
Level 5-9    →  Chick (CHICK)        [Requires Lv5 breakthrough]
Level 10-14  →  Chicken (CHICKEN)     [Requires Lv10 breakthrough]
Level 15-19  →  Big Chicken (BIG_CHICKEN)  [Requires Lv15 breakthrough]
Level 20-25  →  Buff Chicken (BUFF_CHICKEN) [Requires Lv20 breakthrough]
```

### 3. Daily Quest System

| Quest | Goal | Reward |
|-------|------|--------|
| Daily Login | First login of the day | Mood +5, Stamina +10 |
| Exercise 10 Minutes | Accumulate 600 seconds exercise | Strength +20, Stamina +10 |
| Walk 5000 Steps | Accumulate 5000 steps | Strength +50, Mood +10 |

### 4. Map Quest System

**Taipei Landmarks (8 locations):**
1. Taipei 101 - Landmark
2. Elephant Mountain Trail - Hiking trail
3. Daan Forest Park - Urban greenspace
4. Yangmingshan National Park - Natural landscape
5. Beitou Hot Springs - Recreation site
6. Tianmu Sports Park - Sports facility
7. Taipei Arena - Stadium
8. Riverside Bike Path - Exercise route

**Check-in Rewards:**
- Strength +20
- Mood +10
- Records check-in time and location

---

## Deployment and Environment

### Frontend Deployment (Vercel)

```bash
# Install dependencies
npm install

# Local development
npm run dev

# Build production version
npm run build

# Preview production build
npm run preview
```

**Environment Variables:**
```env
VITE_API_URL=https://back-end-production-68f7.up.railway.app
```

### Backend Deployment (Railway)

```bash
# Install dependencies
pip install -r requirements.txt

# Start service
uvicorn app.main:app --host 0.0.0.0 --port 8080

# Reset database (development use)
python reset_database.py
```

**Environment Variables:**
```env
DATABASE_URL=postgresql://username:password@host:port/database
```

### Database Schema

Uses **PostgreSQL** to store all data. Main tables:

```sql
-- Users table
CREATE TABLE users (
    id VARCHAR PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Pets table
CREATE TABLE pets (
    id SERIAL PRIMARY KEY,
    owner_id VARCHAR REFERENCES users(id),
    name VARCHAR DEFAULT '我的手雞',
    strength INTEGER DEFAULT 0,
    stamina INTEGER DEFAULT 900,
    mood INTEGER DEFAULT 0,
    level INTEGER DEFAULT 1,
    stage INTEGER DEFAULT 0,
    daily_exercise_seconds INTEGER DEFAULT 0,
    daily_steps INTEGER DEFAULT 0,
    last_reset_date TIMESTAMP WITH TIME ZONE,
    breakthrough_completed BOOLEAN DEFAULT FALSE,
    last_daily_check TIMESTAMP WITH TIME ZONE
);

-- Exercise logs table
CREATE TABLE exercise_logs (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR REFERENCES users(id),
    pet_id INTEGER REFERENCES pets(id),
    exercise_type VARCHAR,
    duration_seconds INTEGER,
    volume FLOAT,
    steps INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Travel checkins table
CREATE TABLE travel_checkins (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR REFERENCES users(id),
    quest_id VARCHAR,
    completed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    lat FLOAT,
    lng FLOAT
);
```

---

## Development Notes

### 1. CORS Configuration

Backend needs CORS configuration to allow cross-origin requests from frontend:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["*"],
)
```

### 2. GPS Permissions

Frontend requires user to grant location permission:

```typescript
// Check permission status
navigator.permissions.query({ name: 'geolocation' }).then(result => {
  if (result.state === 'granted') {
    // Permission granted
  } else if (result.state === 'prompt') {
    // Need to request permission
  } else if (result.state === 'denied') {
    // Permission denied
  }
});
```

### 3. Developer Mode

Frontend provides developer mode to bypass distance restrictions:

```typescript
const [devMode, setDevMode] = useState(false);

// Use in map component
const isInRange = (quest: QuestLocation): boolean => {
  if (devMode) return true; // Always in range in developer mode
  // ... normal distance checking logic
};
```

### 4. Error Handling Best Practices

```typescript
try {
  const result = await apiFunction();
  // Success handling
  toast.success("Operation successful");
} catch (error) {
  console.error("Error:", error);
  toast.error("Operation failed, please try again later");
}
```

---

## Summary

This project demonstrates the following technical points:

**Frontend Map Implementation**
- Interactive map using Leaflet
- Real-time GPS positioning and tracking
- Geofencing and distance calculation
- Route navigation integration

**Bidirectional Communication**
- RESTful API design
- TypeScript type safety
- React Context state management
- Error handling and user feedback

**Backend Data Processing**
- High-performance FastAPI backend
- SQLAlchemy ORM data modeling
- Complex game logic implementation
- Daily quest and statistics system

**Gamification Design**
- Pet raising system
- Exercise motivation mechanics
- Breakthrough and growth stages
- Location exploration and check-ins

---

**Project Links:**
- Frontend: https://github.com/linskybing/taipei-codefest-1hotcoders/tree/main/frontend
- Backend: https://github.com/linskybing/taipei-codefest-1hotcoders/tree/main/backend
- API Documentation: https://back-end-production-68f7.up.railway.app/docs

**Contact:**
- GitHub: @linskybing
- Project: taipei-codefest-1hotcoders

---

*Last Updated: November 9, 2025*
