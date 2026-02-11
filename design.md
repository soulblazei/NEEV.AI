# Design Document: Anganwadi AI Nutrition and Management System

## Overview

The Anganwadi AI Nutrition and Management System is an offline-first mobile application with cloud backend services designed to support healthcare workers in rural India. The system architecture prioritizes offline operation, data synchronization, and edge computing for ML inference. The design follows a microservices architecture on the backend with a React Native mobile application that maintains a local SQLite database for offline operation.

### Key Design Principles

1. **Offline-First**: All core functionality must work without internet connectivity
2. **Edge Computing**: ML models run on-device for offline predictions
3. **Progressive Enhancement**: Enhanced features when online, core features always available
4. **Data Integrity**: Robust conflict resolution and data validation
5. **Privacy by Design**: Encryption and access control at every layer
6. **Scalability**: Microservices architecture supporting horizontal scaling
7. **Accessibility**: Multi-language support and voice interface for low-literacy users

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Application                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   UI Layer   │  │ Voice Module │  │  CV Module   │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│  ┌──────┴──────────────────┴──────────────────┴───────┐    │
│  │           Business Logic Layer                      │    │
│  │  ┌─────────┐ ┌──────────┐ ┌────────────┐          │    │
│  │  │ Student │ │ Inventory│ │ Prediction │          │    │
│  │  │ Manager │ │ Manager  │ │  Engine    │          │    │
│  │  └─────────┘ └──────────┘ └────────────┘          │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │         Data Layer (SQLite + Sync Engine)           │    │
│  └─────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS/REST API
                            │ (when online)
┌───────────────────────────┴─────────────────────────────────┐
│                      Backend Services                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  API Gateway │  │ Auth Service │  │ Sync Service │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│  ┌──────┴──────────────────┴──────────────────┴───────┐    │
│  │              Microservices Layer                    │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │ Child│   │       ML     │ │ Inventory│           │    │
│  │  │ Service  │ │ Service  │ │ Service  │           │    │
│  │  └──────────┘ └──────────┘ └──────────┘           │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │  Recipe  │ │ Messaging│ │ Analytics│           │    │
│  │  │ Service  │ │ Service  │ │ Service  │           │    │
│  │  └──────────┘ └──────────┘ └──────────┘           │    │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │     Data Layer (PostgreSQL + Redis Cache)           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```


### Technology Stack

**Mobile Application:**
- Framework: React Native (cross-platform iOS/Android)
- Local Database: SQLite with SQLCipher for encryption
- State Management: Redux with Redux Persist
- ML Runtime: TensorFlow Lite for on-device inference
- Computer Vision: TensorFlow Lite + OpenCV
- Voice: React Native Voice with offline speech recognition
- Networking: Axios with offline queue (axios-retry)

**Backend Services:**
- API Framework: Python FastAPI for high-performance async APIs
- ML Framework: PyTorch for training, TensorFlow for serving
- Database: PostgreSQL 14+ with TimescaleDB extension for time-series data
- Cache: Redis for session management and real-time data
- Message Queue: RabbitMQ for async processing
- Object Storage: MinIO (S3-compatible) for images and models
- Container Orchestration: Kubernetes with Helm charts

**ML/AI Pipeline:**
- Training: PyTorch with scikit-learn for preprocessing
- Model Serving: TensorFlow Serving with gRPC
- Feature Store: Feast for feature management
- Experiment Tracking: MLflow
- Model Conversion: ONNX for cross-framework compatibility

**DevOps:**
- CI/CD: GitHub Actions
- Monitoring: Prometheus + Grafana
- Logging: ELK Stack (Elasticsearch, Logstash, Kibana)
- Tracing: Jaeger for distributed tracing

## Components and Interfaces

### Mobile Application Components

#### 1. Student Manager
**Responsibility:** Manage student records, health data, and nutrition tracking

**Key Methods:**
```typescript
interface StudentManager {
  // CRUD operations
  createStudent(data: StudentData): Promise<Student>
  getStudent(id: string): Promise<Student>
  updateStudent(id: string, data: Partial<StudentData>): Promise<Student>
  deleteStudent(id: string): Promise<void>
  
  // Search and filtering
  searchStudents(query: string): Promise<Student[]>
  getStudentsByCenter(centerId: string): Promise<Student[]>
  
  // Health tracking
  addHealthMeasurement(studentId: string, measurement: HealthMeasurement): Promise<void>
  getHealthHistory(studentId: string, days: number): Promise<HealthMeasurement[]>
  
  // Data quality
  detectMissingValues(studentId: string): MissingValueReport
  validateStudentData(data: StudentData): ValidationResult
}
```

#### 2. Prediction Engine
**Responsibility:** Generate nutrition outcome predictions using on-device ML models

**Key Methods:**
```typescript
interface PredictionEngine {
  // Prediction generation
  predictNutritionOutcome(studentId: string, horizonDays: number): Promise<NutritionPrediction>
  batchPredict(studentIds: string[]): Promise<Map<string, NutritionPrediction>>
  
  // Model management
  loadModel(modelPath: string): Promise<void>
  updateModel(newModelData: ArrayBuffer): Promise<void>
  getModelVersion(): string
  
  // Risk assessment
  assessMalnutritionRisk(studentId: string): RiskLevel
  getRiskFactors(studentId: string): RiskFactor[]
}
```

#### 3. Inventory Manager
**Responsibility:** Track food inventory and manage stock levels

**Key Methods:**
```typescript
interface InventoryManager {
  // Inventory operations
  addInventoryItem(item: InventoryItem): Promise<void>
  updateStock(itemId: string, quantity: number): Promise<void>
  getInventory(centerId: string): Promise<InventoryItem[]>
  
  // Computer vision integration
  recognizeItemFromImage(imageData: Blob): Promise<RecognitionResult>
  
  // Stock management
  getLowStockItems(threshold: number): Promise<InventoryItem[]>
  getExpiringItems(daysAhead: number): Promise<InventoryItem[]>
  recordConsumption(itemId: string, quantity: number, date: Date): Promise<void>
}
```

#### 4. Recipe Engine
**Responsibility:** Generate personalized recipe suggestions

**Key Methods:**
```typescript
interface RecipeEngine {
  // Recipe suggestions
  suggestRecipes(studentId: string, count: number): Promise<Recipe[]>
  getRecipesByDeficiency(deficiency: NutritionalDeficiency): Promise<Recipe[]>
  
  // Recipe management
  getRecipe(id: string): Promise<Recipe>
  searchRecipes(query: string, filters: RecipeFilters): Promise<Recipe[]>
  
  // Meal tracking
  recordMealPrepared(studentId: string, recipeId: string, date: Date): Promise<void>
  getMealHistory(studentId: string, days: number): Promise<MealRecord[]>
}
```

#### 5. Sync Engine
**Responsibility:** Synchronize local data with backend when online

**Key Methods:**
```typescript
interface SyncEngine {
  // Sync operations
  syncAll(): Promise<SyncResult>
  syncStudents(): Promise<SyncResult>
  syncInventory(): Promise<SyncResult>
  syncMessages(): Promise<SyncResult>
  
  // Conflict resolution
  resolveConflict(conflict: DataConflict, resolution: ConflictResolution): Promise<void>
  getPendingConflicts(): Promise<DataConflict[]>
  
  // Status monitoring
  getSyncStatus(): SyncStatus
  getPendingChanges(): PendingChange[]
  getLastSyncTime(): Date
}
```

#### 6. Voice Interface
**Responsibility:** Handle voice input and output in vernacular languages

**Key Methods:**
```typescript
interface VoiceInterface {
  // Speech recognition
  startListening(language: string): Promise<void>
  stopListening(): Promise<string>
  recognizeSpeech(audioData: Blob, language: string): Promise<string>
  
  // Speech synthesis
  speak(text: string, language: string): Promise<void>
  stopSpeaking(): void
  
  // Language management
  getSupportedLanguages(): Language[]
  setLanguage(language: string): void
  getLanguage(): string
}
```

#### 7. Computer Vision Module
**Responsibility:** Recognize inventory items from images

**Key Methods:**
```typescript
interface CVModule {
  // Image recognition
  recognizeItem(imageData: Blob): Promise<RecognitionResult>
  batchRecognize(images: Blob[]): Promise<RecognitionResult[]>
  
  // Model management
  loadCVModel(modelPath: string): Promise<void>
  updateCVModel(newModelData: ArrayBuffer): Promise<void>
  
  // Preprocessing
  preprocessImage(imageData: Blob): Promise<Blob>
  validateImageQuality(imageData: Blob): ImageQualityReport
}
```

#### 8. Messaging Service
**Responsibility:** Handle communication between healthcare workers

**Key Methods:**
```typescript
interface MessagingService {
  // Message operations
  sendMessage(recipientId: string, content: MessageContent): Promise<Message>
  getMessages(conversationId: string): Promise<Message[]>
  markAsRead(messageId: string): Promise<void>
  
  // Referrals
  createReferral(studentId: string, recipientId: string, notes: string): Promise<Referral>
  getReferrals(status: ReferralStatus): Promise<Referral[]>
  updateReferralStatus(referralId: string, status: ReferralStatus): Promise<void>
  
  // Notifications
  getNotifications(): Promise<Notification[]>
  dismissNotification(notificationId: string): Promise<void>
}
```


### Backend Service Components

#### 1. Student Service
**Responsibility:** Manage student records and health data in the cloud

**API Endpoints:**
```
POST   /api/v1/students                    # Create student
GET    /api/v1/students/{id}               # Get student by ID
PUT    /api/v1/students/{id}               # Update student
DELETE /api/v1/students/{id}               # Delete student
GET    /api/v1/students/center/{centerId}  # Get students by center
POST   /api/v1/students/search             # Search students
GET    /api/v1/students/{id}/health        # Get health history
POST   /api/v1/students/{id}/health        # Add health measurement
```

#### 2. ML Service
**Responsibility:** Train and serve ML models for predictions and data quality

**API Endpoints:**
```
POST   /api/v1/ml/predict                  # Generate prediction
POST   /api/v1/ml/batch-predict            # Batch predictions
POST   /api/v1/ml/detect-ghosts            # Detect ghost entries
POST   /api/v1/ml/detect-missing           # Detect missing values
GET    /api/v1/ml/models/{type}/latest     # Get latest model version
POST   /api/v1/ml/models/train             # Trigger model training
```

**ML Models:**

1. **Nutrition Outcome Predictor**
   - Type: Gradient Boosting (XGBoost/LightGBM)
   - Input Features: age, weight, height, BMI, weight_velocity, height_velocity, attendance_rate, meal_consumption_rate, historical_measurements (time series)
   - Output: Predicted weight, height, BMI at 30/60/90 days + confidence intervals
   - Training: Weekly retraining on new data

2. **Ghost Entry Detector**
   - Type: Ensemble (Random Forest + Fuzzy Matching)
   - Input Features: name_similarity, dob_similarity, parent_name_similarity, address_similarity, biometric_match, registration_pattern
   - Output: Duplicate probability score (0-1) + matched record IDs
   - Training: Monthly retraining with labeled duplicates

3. **Missing Value Imputation**
   - Type: K-Nearest Neighbors + MICE (Multiple Imputation by Chained Equations)
   - Input: Partial student record
   - Output: Imputed values with confidence scores
   - Training: Quarterly retraining

4. **Inventory Item Recognizer**
   - Type: Convolutional Neural Network (MobileNetV3 or EfficientNet-Lite)
   - Input: Image (224x224 RGB)
   - Output: Item class + confidence score
   - Training: Continuous learning with user corrections

#### 3. Inventory Service
**Responsibility:** Manage inventory data and consumption tracking

**API Endpoints:**
```
POST   /api/v1/inventory                   # Add inventory item
GET    /api/v1/inventory/center/{centerId} # Get inventory by center
PUT    /api/v1/inventory/{id}              # Update inventory item
POST   /api/v1/inventory/{id}/consume      # Record consumption
GET    /api/v1/inventory/low-stock         # Get low stock items
GET    /api/v1/inventory/expiring          # Get expiring items
```

#### 4. Recipe Service
**Responsibility:** Manage recipes and generate personalized suggestions

**API Endpoints:**
```
GET    /api/v1/recipes                     # List recipes
GET    /api/v1/recipes/{id}                # Get recipe by ID
POST   /api/v1/recipes/suggest             # Get personalized suggestions
GET    /api/v1/recipes/deficiency/{type}   # Get recipes for deficiency
POST   /api/v1/recipes/{id}/prepared       # Record meal prepared
```

**Recipe Recommendation Algorithm:**
1. Identify nutritional deficiencies from student health data
2. Query recipes that address those deficiencies
3. Filter by available inventory items
4. Rank by: deficiency coverage (40%), ingredient availability (30%), cultural preference (20%), preparation complexity (10%)
5. Return top N recipes with explanations

#### 5. Messaging Service
**Responsibility:** Handle inter-worker communication and referrals

**API Endpoints:**
```
POST   /api/v1/messages                    # Send message
GET    /api/v1/messages/conversation/{id}  # Get conversation messages
PUT    /api/v1/messages/{id}/read          # Mark as read
POST   /api/v1/referrals                   # Create referral
GET    /api/v1/referrals                   # Get referrals
PUT    /api/v1/referrals/{id}              # Update referral status
GET    /api/v1/notifications               # Get notifications
```

#### 6. Sync Service
**Responsibility:** Handle data synchronization and conflict resolution

**API Endpoints:**
```
POST   /api/v1/sync/pull                   # Pull changes from server
POST   /api/v1/sync/push                   # Push changes to server
POST   /api/v1/sync/resolve-conflict       # Resolve sync conflict
GET    /api/v1/sync/status                 # Get sync status
```

**Sync Protocol:**
1. Client sends last sync timestamp + pending changes
2. Server identifies conflicts (same record modified on server after client's last sync)
3. Server applies non-conflicting changes
4. Server returns conflicts + server changes since last sync
5. Client resolves conflicts (last-write-wins with user notification)
6. Client sends conflict resolutions
7. Server applies resolutions and confirms

#### 7. Analytics Service
**Responsibility:** Generate reports and dashboards

**API Endpoints:**
```
GET    /api/v1/analytics/malnutrition-rate # Get malnutrition statistics
GET    /api/v1/analytics/data-quality      # Get data quality metrics
GET    /api/v1/analytics/predictions       # Get prediction accuracy
POST   /api/v1/analytics/reports           # Generate custom report
GET    /api/v1/analytics/trends            # Get nutrition trends
```

#### 8. Auth Service
**Responsibility:** Handle authentication and authorization

**API Endpoints:**
```
POST   /api/v1/auth/login                  # Login with credentials
POST   /api/v1/auth/logout                 # Logout
POST   /api/v1/auth/refresh                # Refresh access token
POST   /api/v1/auth/verify-otp             # Verify OTP for MFA
GET    /api/v1/auth/permissions            # Get user permissions
```


## Data Models

### Student Record

```typescript
interface Student {
  id: string                          // UUID
  centerId: string                    // Anganwadi center ID
  personalInfo: {
    firstName: string
    lastName: string
    dateOfBirth: Date
    gender: 'male' | 'female' | 'other'
    bloodGroup?: string
    aadharNumber?: string             // Encrypted
    photographUrl?: string
  }
  parentInfo: {
    motherName: string
    fatherName: string
    guardianName?: string
    contactNumber: string             // Encrypted
    address: string
  }
  healthData: {
    currentWeight: number             // kg
    currentHeight: number             // cm
    currentBMI: number
    allergies: string[]
    medicalConditions: string[]
    vaccinationStatus: VaccinationRecord[]
  }
  nutritionData: {
    deficiencies: NutritionalDeficiency[]
    mealHistory: MealRecord[]
    supplementsGiven: SupplementRecord[]
  }
  attendance: {
    totalDays: number
    presentDays: number
    attendanceRate: number
  }
  metadata: {
    createdAt: Date
    updatedAt: Date
    createdBy: string                 // Healthcare worker ID
    lastModifiedBy: string
    syncStatus: 'synced' | 'pending' | 'conflict'
    version: number                   // For optimistic locking
  }
}

interface HealthMeasurement {
  id: string
  studentId: string
  measurementDate: Date
  weight: number                      // kg
  height: number                      // cm
  bmi: number
  muac?: number                       // Mid-upper arm circumference (cm)
  headCircumference?: number          // cm (for infants)
  notes?: string
  measuredBy: string                  // Healthcare worker ID
  createdAt: Date
}

interface NutritionalDeficiency {
  type: 'protein' | 'iron' | 'vitamin_a' | 'vitamin_d' | 'calcium' | 'zinc' | 'iodine'
  severity: 'mild' | 'moderate' | 'severe'
  detectedDate: Date
  status: 'active' | 'improving' | 'resolved'
}

interface MealRecord {
  id: string
  studentId: string
  recipeId: string
  recipeName: string
  date: Date
  portionSize: number                 // grams
  consumed: boolean
  consumptionPercentage?: number      // 0-100
  recordedBy: string
}
```

### Inventory Item

```typescript
interface InventoryItem {
  id: string
  centerId: string
  itemInfo: {
    name: string
    category: 'grain' | 'pulse' | 'vegetable' | 'fruit' | 'dairy' | 'oil' | 'spice' | 'other'
    unit: 'kg' | 'liter' | 'piece' | 'packet'
    imageUrl?: string
  }
  stock: {
    currentQuantity: number
    minimumThreshold: number
    maximumCapacity: number
    reorderQuantity: number
  }
  tracking: {
    expirationDate?: Date
    batchNumber?: string
    supplier?: string
    costPerUnit?: number
  }
  consumption: {
    dailyAverageUsage: number
    lastConsumedDate?: Date
    totalConsumed: number
  }
  metadata: {
    createdAt: Date
    updatedAt: Date
    lastStockUpdate: Date
    syncStatus: 'synced' | 'pending' | 'conflict'
  }
}

interface ConsumptionRecord {
  id: string
  itemId: string
  quantity: number
  date: Date
  purpose: 'meal_preparation' | 'wastage' | 'distribution' | 'other'
  notes?: string
  recordedBy: string
}
```

### Recipe

```typescript
interface Recipe {
  id: string
  name: string
  description: string
  regionalVariant?: string            // e.g., "Tamil Nadu", "Bihar"
  category: 'breakfast' | 'lunch' | 'dinner' | 'snack'
  ageGroup: {
    minAge: number                    // months
    maxAge: number                    // months
  }
  ingredients: Ingredient[]
  preparation: {
    steps: string[]
    preparationTime: number           // minutes
    cookingTime: number               // minutes
    difficulty: 'easy' | 'medium' | 'hard'
  }
  nutrition: {
    calories: number                  // per serving
    protein: number                   // grams
    carbohydrates: number             // grams
    fat: number                       // grams
    fiber: number                     // grams
    vitamins: { [key: string]: number }
    minerals: { [key: string]: number }
  }
  addresses: {
    deficiencies: NutritionalDeficiency['type'][]
    healthBenefits: string[]
  }
  metadata: {
    createdAt: Date
    updatedAt: Date
    popularity: number                // usage count
    rating?: number                   // 1-5
  }
}

interface Ingredient {
  itemId: string
  itemName: string
  quantity: number
  unit: string
  optional: boolean
  substitutes?: string[]
}
```

### Prediction

```typescript
interface NutritionPrediction {
  id: string
  studentId: string
  predictionDate: Date
  predictions: {
    horizon30Days: PredictionPoint
    horizon60Days: PredictionPoint
    horizon90Days: PredictionPoint
  }
  riskAssessment: {
    overallRisk: 'low' | 'medium' | 'high'
    riskFactors: RiskFactor[]
    recommendations: string[]
  }
  modelInfo: {
    modelVersion: string
    confidence: number                // 0-1
    featureImportance: { [feature: string]: number }
  }
  metadata: {
    createdAt: Date
    generatedBy: 'mobile' | 'server'
  }
}

interface PredictionPoint {
  predictedWeight: number
  predictedHeight: number
  predictedBMI: number
  confidenceInterval: {
    weightLower: number
    weightUpper: number
    heightLower: number
    heightUpper: number
  }
  malnutritionProbability: number     // 0-1
}

interface RiskFactor {
  factor: string
  impact: 'high' | 'medium' | 'low'
  description: string
  actionable: boolean
}
```

### Message and Referral

```typescript
interface Message {
  id: string
  conversationId: string
  senderId: string
  recipientId: string
  content: {
    type: 'text' | 'voice' | 'image' | 'referral'
    text?: string
    voiceUrl?: string
    imageUrl?: string
    referralId?: string
  }
  metadata: {
    sentAt: Date
    deliveredAt?: Date
    readAt?: Date
    encrypted: boolean
    syncStatus: 'synced' | 'pending'
  }
}

interface Referral {
  id: string
  studentId: string
  fromWorkerId: string
  toWorkerId: string
  referralType: 'medical_checkup' | 'nutrition_counseling' | 'vaccination' | 'emergency' | 'other'
  priority: 'low' | 'medium' | 'high' | 'urgent'
  details: {
    reason: string
    symptoms?: string[]
    attachments?: string[]            // URLs to images/documents
    healthSummary: {
      currentWeight: number
      currentHeight: number
      recentMeasurements: HealthMeasurement[]
      activeDeficiencies: NutritionalDeficiency[]
    }
  }
  status: 'pending' | 'accepted' | 'in_progress' | 'completed' | 'rejected'
  timeline: {
    createdAt: Date
    acceptedAt?: Date
    completedAt?: Date
    notes?: string
  }
}
```

### User and Authentication

```typescript
interface User {
  id: string
  personalInfo: {
    firstName: string
    lastName: string
    phoneNumber: string               // Encrypted
    email?: string
    photographUrl?: string
  }
  role: 'anganwadi_worker' | 'asha_worker' | 'anm' | 'supervisor' | 'admin'
  assignment: {
    centerId?: string                 // For Anganwadi workers
    district: string
    block: string
    assignedCenters: string[]         // For supervisors/ANMs
  }
  preferences: {
    language: string
    voiceEnabled: boolean
    notificationSettings: NotificationSettings
  }
  authentication: {
    passwordHash: string
    mfaEnabled: boolean
    mfaSecret?: string
    lastLogin?: Date
    failedLoginAttempts: number
  }
  permissions: Permission[]
  metadata: {
    createdAt: Date
    updatedAt: Date
    status: 'active' | 'inactive' | 'suspended'
  }
}

interface Permission {
  resource: string                    // e.g., 'students', 'inventory', 'reports'
  actions: ('create' | 'read' | 'update' | 'delete')[]
  scope: 'own' | 'center' | 'block' | 'district' | 'all'
}
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Ghost Entry Detection Properties

**Property 1: Duplicate detection identifies similar records**
*For any* set of student records with matching demographic data (name, DOB, parent info) within the same Anganwadi center, the ghost detection algorithm should flag these records as potential duplicates with a similarity score above the configured threshold.
**Validates: Requirements 1.1, 1.2**

**Property 2: Biometric matching increases confidence**
*For any* pair of student records where one has only demographic matches and another has both demographic and biometric matches, the biometric-matched pair should have a higher duplicate confidence score.
**Validates: Requirements 1.3**

**Property 3: Ghost detection generates complete reports**
*For any* detected ghost entry, the generated report should contain all suspicious record IDs, their similarity scores, and the specific matching fields that triggered the detection.
**Validates: Requirements 1.4**

**Property 4: Ghost actions are audited**
*For any* ghost entry detection or resolution action (merge/delete), there should exist a corresponding audit log entry containing timestamp, user ID, action type, and affected record IDs.
**Validates: Requirements 1.6**

### Missing Value Detection Properties

**Property 5: Missing value detection is complete**
*For any* student record, the missing value detection function should identify and return all fields that are null, undefined, or empty strings.
**Validates: Requirements 2.1**

**Property 6: Critical missing values prevent predictions**
*For any* student record with one or more missing critical health metrics (weight, height, or age), attempting to generate a nutrition prediction should either fail with an error or mark the record as incomplete and return no prediction.
**Validates: Requirements 2.2**

**Property 7: Non-critical missing values allow predictions with warnings**
*For any* student record with complete critical fields but missing non-critical fields, the prediction generation should succeed and the result should include a data quality warning flag.
**Validates: Requirements 2.3**

**Property 8: Missing fields are highlighted in UI**
*For any* student record with one or more missing values, the UI rendering function should produce output that includes visual indicators (highlighting, icons, or markers) for each missing field.
**Validates: Requirements 2.4**

**Property 9: Sync prioritizes complete records**
*For any* set of pending records to sync, when ordered by the sync priority algorithm, records with complete critical data should appear before records with missing critical data in the transmission queue.
**Validates: Requirements 2.6**

### Nutrition Prediction Properties

**Property 10: Predictions cover all time horizons**
*For any* student record with sufficient historical data (at least 3 measurements over 30 days), the prediction service should generate nutrition outcome predictions for all three time horizons: 30, 60, and 90 days.
**Validates: Requirements 3.1**

**Property 11: Risk classification is valid**
*For any* nutrition prediction generated by the system, the risk level field should contain exactly one of the valid enumerated values: 'low', 'medium', or 'high'.
**Validates: Requirements 3.3**

**Property 12: High-risk predictions trigger alerts**
*For any* nutrition prediction with risk level 'high', the mobile app should generate and display an alert object containing the prediction details and recommended interventions.
**Validates: Requirements 3.4**

### Offline Operation Properties

**Property 13: Offline changes are marked pending**
*For any* create or update operation performed when the mobile app is in offline mode (no network connectivity), the resulting record should have its syncStatus field set to 'pending'.
**Validates: Requirements 4.2**

**Property 14: Sync transmits all pending changes**
*For any* set of records with syncStatus 'pending' before a sync operation, after the sync completes successfully, all those records should have syncStatus changed to 'synced' (assuming no conflicts).
**Validates: Requirements 4.4**

**Property 15: Conflict resolution applies last-write-wins**
*For any* sync conflict where the same record was modified both offline and online, the conflict resolution algorithm should retain the version with the later updatedAt timestamp and generate a conflict notification for the user.
**Validates: Requirements 4.5**

**Property 16: Storage eviction retains recent records**
*For any* storage eviction operation triggered when local capacity is exceeded, the set of records retained in storage should be those with the most recent lastAccessedAt timestamps, up to the storage limit.
**Validates: Requirements 4.7**

### Voice Interface Properties

**Property 17: Voice feedback uses selected language**
*For any* alert or feedback message when voice synthesis is enabled, the generated speech audio should be in the language specified by the healthcare worker's current language preference setting.
**Validates: Requirements 5.5**

**Property 18: Low-confidence recognition prompts correction**
*For any* voice recognition result with confidence score below 70%, the mobile app should display the recognized text in an editable field allowing manual correction before submission.
**Validates: Requirements 5.7**

### Computer Vision Inventory Properties

**Property 19: Recognized items prompt confirmation**
*For any* inventory item successfully recognized from an image (confidence >= 70%), the mobile app should display a confirmation prompt showing the recognized item name and request quantity input before updating the inventory database.
**Validates: Requirements 6.3**

**Property 20: Low-confidence recognition allows manual selection**
*For any* CV recognition result with confidence score below 70%, the mobile app should provide a manual item selection interface (dropdown, search, or list) allowing the user to choose the correct item.
**Validates: Requirements 6.4**

**Property 21: Inventory tracking is comprehensive**
*For any* inventory item record in the database, the record should contain non-null values for current quantity, consumption rate (calculated or default), and expiration date (if applicable to the item category).
**Validates: Requirements 6.6**

**Property 22: Low stock triggers alerts**
*For any* inventory item where the current quantity value falls below the configured minimum threshold value for that item, the system should generate a low-stock alert notification.
**Validates: Requirements 6.7**

### Recipe Suggestion Properties

**Property 23: Recipe suggestions address deficiencies**
*For any* student with one or more identified nutritional deficiencies, every recipe in the suggested recipes list should address at least one of those specific deficiencies (i.e., the recipe's addresses.deficiencies array should contain at least one deficiency type that matches the student's deficiencies).
**Validates: Requirements 7.1**

**Property 24: Recipe suggestions respect allergy constraints**
*For any* student with documented allergies, all suggested recipes should not contain any ingredients that match those allergens (i.e., no recipe ingredient should appear in the student's allergy list).
**Validates: Requirements 7.2**

**Property 25: Recipe suggestions use available ingredients**
*For any* suggested recipe, all required (non-optional) ingredients should be present in the current local inventory with quantity greater than or equal to the recipe's required quantity.
**Validates: Requirements 7.3**

**Property 26: Recipe display is complete**
*For any* recipe displayed in the mobile app UI, the rendered output should include all of the following fields: ingredients list, preparation steps, portion sizes, and nutritional benefits.
**Validates: Requirements 7.5**

**Property 27: Meal preparation is logged**
*For any* meal preparation action (marking a recipe as prepared for a student), the system should create a new meal record entry in the database and update the student's nutrition tracking data with the meal information.
**Validates: Requirements 7.6**

**Property 28: Multi-deficiency recipes rank higher**
*For any* two recipes in a suggestion list where one recipe addresses N deficiencies and another addresses M deficiencies (where N > M), the recipe addressing more deficiencies should have a higher ranking score and appear earlier in the sorted suggestion list.
**Validates: Requirements 7.7**

### Interconnection System Properties

**Property 29: Referrals include health summary**
*For any* referral created by the system, the referral object should include a health summary containing the student's current weight, current height, recent measurements (last 3-5 measurements), and all active nutritional deficiencies.
**Validates: Requirements 8.1**

**Property 30: Referrals trigger notifications**
*For any* referral created in the system, a notification should be generated and queued for delivery to the recipient healthcare worker specified in the referral.
**Validates: Requirements 8.2**

**Property 31: Offline messages are queued**
*For any* message sent when the mobile app is in offline mode, the message should be stored locally with syncStatus 'pending' and transmitted to the server when connectivity is restored.
**Validates: Requirements 8.3**

**Property 32: Messages are encrypted**
*For any* message stored in the database or transmitted over the network, the message content should be encrypted using the system's encryption algorithm (AES-256).
**Validates: Requirements 8.5**

**Property 33: Message history respects authorization**
*For any* request to access a student's message history, the system should verify that the requesting healthcare worker has authorization for that student before returning the messages.
**Validates: Requirements 8.6**

**Property 34: Critical alerts notify care network**
*For any* critical health alert detected for a student, notifications should be generated and sent to all healthcare workers who are members of that student's care network.
**Validates: Requirements 8.7**

**Property 35: Inbox organizes by student**
*For any* inbox view rendered in the mobile app, all messages, referrals, and alerts should be grouped by their associated student ID, with each group clearly separated.
**Validates: Requirements 8.8**

### Security and Privacy Properties

**Property 36: Student records are encrypted**
*For any* student record stored in the local database or transmitted over the network, the sensitive data fields should be encrypted using AES-256 encryption before storage or transmission.
**Validates: Requirements 9.1**

**Property 37: Role-based permissions are enforced**
*For any* user action attempting to access or modify a resource, the system should verify that the user's role includes the required permission for that action and resource before allowing the operation to proceed.
**Validates: Requirements 9.3**

**Property 38: Record access is audited**
*For any* student record access operation (read, update, delete), an audit log entry should be created and persisted containing the timestamp, user ID, action type, and record ID.
**Validates: Requirements 9.4**

**Property 39: Local data is encrypted**
*For any* data stored in the mobile app's local SQLite database, the data should be encrypted using device-level encryption (SQLCipher) with a secure encryption key.
**Validates: Requirements 9.6**

### Reporting and Analytics Properties

**Property 40: Reports include data quality metrics**
*For any* report generated by the system, the report output should include data quality metrics sections showing: number of ghost entries detected, count of missing values by field, and data completeness percentages by center.
**Validates: Requirements 10.2**

**Property 41: Threshold alerts are generated**
*For any* Anganwadi center where the calculated malnutrition rate exceeds the configured threshold value, an automated alert should be generated and sent to the appropriate administrators.
**Validates: Requirements 10.6**

**Property 42: Comparisons include statistical significance**
*For any* time period comparison displayed in reports (e.g., comparing malnutrition rates between two months), the system should calculate and display the statistical significance (p-value) of the observed changes in nutrition outcomes.
**Validates: Requirements 10.7**

### User Experience Properties

**Property 43: Help tooltips are available**
*For any* major feature in the mobile app (as defined in the feature list), a contextual help tooltip should be available and accessible to the user through a help icon or gesture.
**Validates: Requirements 12.2**

**Property 44: Error messages include solutions**
*For any* error message displayed to the user, the error message text should include a suggested solution or next step that the user can take to resolve or work around the error.
**Validates: Requirements 12.4**

**Property 45: New features show introductions**
*For any* feature marked as "new" that is accessed for the first time by a user (tracked by user preferences), the mobile app should display a brief introduction or tutorial overlay before allowing full interaction with the feature.
**Validates: Requirements 12.7**


## Error Handling

### Error Categories

**1. Network Errors**
- Connection timeout
- Server unavailable
- DNS resolution failure
- SSL/TLS errors

**Handling Strategy:**
- Graceful degradation to offline mode
- Queue operations for later sync
- Display user-friendly message: "Working offline. Changes will sync when connected."
- Retry with exponential backoff (1s, 2s, 4s, 8s, max 30s)

**2. Data Validation Errors**
- Invalid input format
- Missing required fields
- Out-of-range values
- Constraint violations

**Handling Strategy:**
- Validate on client before submission
- Display inline validation errors with specific field highlighting
- Provide clear error messages: "Weight must be between 1 and 50 kg"
- Prevent form submission until errors are resolved
- Log validation errors for analytics

**3. Sync Conflicts**
- Same record modified offline and online
- Deleted record modified offline
- Version mismatch

**Handling Strategy:**
- Apply last-write-wins based on timestamp
- Notify user of conflict with details
- Provide option to view both versions
- Log all conflicts for administrator review
- Maintain conflict history for 30 days

**4. ML Model Errors**
- Model file corrupted
- Insufficient data for prediction
- Model inference timeout
- Unsupported input format

**Handling Strategy:**
- Fall back to rule-based predictions if model fails
- Display warning: "Prediction may be less accurate"
- Log model errors for monitoring
- Attempt model re-download on next sync
- Provide manual override option for healthcare workers

**5. Computer Vision Errors**
- Poor image quality
- Unrecognized item
- Multiple items in frame
- Insufficient lighting

**Handling Strategy:**
- Provide real-time feedback during capture: "Move closer" or "Improve lighting"
- Allow manual item selection if recognition fails
- Save failed images for model retraining
- Display confidence score to user
- Suggest retaking photo if confidence < 50%

**6. Voice Recognition Errors**
- Ambient noise interference
- Unsupported accent/dialect
- Low confidence recognition
- Microphone permission denied

**Handling Strategy:**
- Display recognized text for confirmation
- Provide edit option for corrections
- Fall back to text input if recognition repeatedly fails
- Show noise level indicator during recording
- Request microphone permission with clear explanation

**7. Storage Errors**
- Disk full
- Database corruption
- Encryption key loss
- File system errors

**Handling Strategy:**
- Monitor storage usage and warn at 80% capacity
- Implement automatic cleanup of old cached data
- Maintain database backups
- Provide database repair utility
- Escalate critical errors to support team

**8. Authentication Errors**
- Invalid credentials
- Expired session
- MFA failure
- Account locked

**Handling Strategy:**
- Clear error messages without revealing security details
- Implement account lockout after 5 failed attempts
- Provide password reset flow
- Log all authentication failures
- Notify user of suspicious login attempts

### Error Logging and Monitoring

**Client-Side Logging:**
- Log level: ERROR, WARN, INFO, DEBUG
- Include: timestamp, user ID, device info, app version, error stack trace
- Store locally and sync to server when online
- Implement log rotation (max 10 MB per log file)
- Sanitize PII before logging

**Server-Side Monitoring:**
- Centralized logging with ELK stack
- Real-time alerting for critical errors (>10 errors/minute)
- Error rate dashboards by service and endpoint
- Distributed tracing for request flows
- Weekly error trend reports

## Testing Strategy

### Dual Testing Approach

The system requires both **unit testing** and **property-based testing** for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs
- Both approaches are complementary and necessary

### Unit Testing

**Focus Areas:**
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, null handling)
- Error conditions and exception handling
- Integration points between components
- UI component rendering and interactions

**Example Unit Tests:**
```typescript
// Example: Test specific ghost detection case
test('detects duplicate students with same name and DOB', () => {
  const student1 = createStudent({ name: 'Raj Kumar', dob: '2020-01-15' })
  const student2 = createStudent({ name: 'Raj Kumar', dob: '2020-01-15' })
  const result = detectGhosts([student1, student2])
  expect(result.duplicates).toHaveLength(1)
  expect(result.duplicates[0].similarityScore).toBeGreaterThan(0.8)
})

// Example: Test edge case for missing values
test('handles student record with all fields missing', () => {
  const emptyStudent = { id: '123' }
  const missing = detectMissingValues(emptyStudent)
  expect(missing.criticalFields).toContain('weight')
  expect(missing.criticalFields).toContain('height')
  expect(missing.criticalFields).toContain('dateOfBirth')
})

// Example: Test error condition
test('prediction fails gracefully with insufficient data', () => {
  const studentWithNoHistory = createStudent({ measurements: [] })
  expect(() => predictNutrition(studentWithNoHistory))
    .toThrow('Insufficient historical data for prediction')
})
```

**Unit Test Configuration:**
- Framework: Jest for JavaScript/TypeScript, pytest for Python
- Coverage target: 80% line coverage minimum
- Run on every commit via CI/CD
- Mock external dependencies (API calls, database, ML models)

### Property-Based Testing

**Focus Areas:**
- Universal properties that hold for all inputs
- Invariants that must be maintained
- Round-trip properties (serialize/deserialize, encrypt/decrypt)
- Metamorphic properties (relationships between inputs/outputs)
- Comprehensive input coverage through randomization

**Property Test Configuration:**
- Framework: fast-check for JavaScript/TypeScript, Hypothesis for Python
- Minimum 100 iterations per property test (due to randomization)
- Each test must reference its design document property
- Tag format: **Feature: anganwadi-ai-nutrition-system, Property {number}: {property_text}**

**Example Property Tests:**
```typescript
// Feature: anganwadi-ai-nutrition-system, Property 1: Duplicate detection identifies similar records
test('ghost detection flags records with matching demographics', () => {
  fc.assert(
    fc.property(
      fc.array(studentGenerator(), { minLength: 2, maxLength: 100 }),
      (students) => {
        // Create a duplicate by copying a random student
        const duplicate = { ...students[0], id: generateId() }
        const allStudents = [...students, duplicate]
        
        const result = detectGhosts(allStudents)
        
        // Property: Should detect the duplicate
        const foundDuplicate = result.duplicates.some(d => 
          d.records.includes(students[0].id) && 
          d.records.includes(duplicate.id)
        )
        expect(foundDuplicate).toBe(true)
      }
    ),
    { numRuns: 100 }
  )
})

// Feature: anganwadi-ai-nutrition-system, Property 14: Sync transmits all pending changes
test('sync changes all pending records to synced', () => {
  fc.assert(
    fc.property(
      fc.array(studentGenerator(), { minLength: 1, maxLength: 50 }),
      async (students) => {
        // Mark all as pending
        const pendingStudents = students.map(s => ({ 
          ...s, 
          metadata: { ...s.metadata, syncStatus: 'pending' } 
        }))
        
        await syncEngine.syncAll(pendingStudents)
        
        // Property: All should now be synced
        const allSynced = pendingStudents.every(s => 
          s.metadata.syncStatus === 'synced'
        )
        expect(allSynced).toBe(true)
      }
    ),
    { numRuns: 100 }
  )
})

// Feature: anganwadi-ai-nutrition-system, Property 23: Recipe suggestions address deficiencies
test('all suggested recipes address student deficiencies', () => {
  fc.assert(
    fc.property(
      studentWithDeficienciesGenerator(),
      fc.array(recipeGenerator(), { minLength: 10, maxLength: 100 }),
      (student, recipeDatabase) => {
        const suggestions = suggestRecipes(student, recipeDatabase, 5)
        
        // Property: Every suggested recipe should address at least one deficiency
        const allAddressDeficiencies = suggestions.every(recipe =>
          recipe.addresses.deficiencies.some(d =>
            student.nutritionData.deficiencies.map(sd => sd.type).includes(d)
          )
        )
        expect(allAddressDeficiencies).toBe(true)
      }
    ),
    { numRuns: 100 }
  )
})
```

**Generators for Property Tests:**
```typescript
// Generator for random student records
const studentGenerator = () => fc.record({
  id: fc.uuid(),
  centerId: fc.uuid(),
  personalInfo: fc.record({
    firstName: fc.string({ minLength: 2, maxLength: 20 }),
    lastName: fc.string({ minLength: 2, maxLength: 20 }),
    dateOfBirth: fc.date({ min: new Date('2015-01-01'), max: new Date('2023-12-31') }),
    gender: fc.constantFrom('male', 'female', 'other')
  }),
  healthData: fc.record({
    currentWeight: fc.float({ min: 5, max: 30, noNaN: true }),
    currentHeight: fc.float({ min: 50, max: 120, noNaN: true }),
    currentBMI: fc.float({ min: 10, max: 25, noNaN: true })
  }),
  metadata: fc.record({
    syncStatus: fc.constantFrom('synced', 'pending', 'conflict'),
    version: fc.nat()
  })
})

// Generator for students with deficiencies
const studentWithDeficienciesGenerator = () => studentGenerator().chain(student =>
  fc.record({
    ...student,
    nutritionData: fc.record({
      deficiencies: fc.array(
        fc.record({
          type: fc.constantFrom('protein', 'iron', 'vitamin_a', 'vitamin_d', 'calcium'),
          severity: fc.constantFrom('mild', 'moderate', 'severe'),
          status: fc.constant('active')
        }),
        { minLength: 1, maxLength: 3 }
      )
    })
  })
)

// Generator for recipes
const recipeGenerator = () => fc.record({
  id: fc.uuid(),
  name: fc.string({ minLength: 5, maxLength: 30 }),
  ingredients: fc.array(
    fc.record({
      itemId: fc.uuid(),
      itemName: fc.string(),
      quantity: fc.nat({ max: 500 }),
      optional: fc.boolean()
    }),
    { minLength: 3, maxLength: 10 }
  ),
  addresses: fc.record({
    deficiencies: fc.array(
      fc.constantFrom('protein', 'iron', 'vitamin_a', 'vitamin_d', 'calcium'),
      { minLength: 1, maxLength: 3 }
    )
  })
})
```

### Integration Testing

**Focus Areas:**
- End-to-end workflows (student registration → measurement → prediction → recipe suggestion)
- API contract testing between mobile app and backend
- Database integration and transaction handling
- ML model integration and inference
- Sync engine with conflict scenarios

**Tools:**
- Detox for mobile app E2E testing
- Postman/Newman for API testing
- Docker Compose for local integration environment

### Performance Testing

**Focus Areas:**
- Mobile app responsiveness (< 2s for data retrieval)
- ML inference time (< 5s for predictions)
- Sync performance with large datasets
- Database query optimization
- API endpoint latency

**Tools:**
- Lighthouse for mobile performance
- k6 or Locust for load testing
- Profiling tools (React DevTools, Python cProfile)

### Security Testing

**Focus Areas:**
- Penetration testing for API endpoints
- Encryption verification (at rest and in transit)
- Authentication and authorization testing
- SQL injection and XSS prevention
- Sensitive data exposure

**Tools:**
- OWASP ZAP for security scanning
- Burp Suite for penetration testing
- npm audit / pip-audit for dependency vulnerabilities

### Accessibility Testing

**Focus Areas:**
- Screen reader compatibility
- Voice interface usability
- Color contrast and font sizes
- Touch target sizes (minimum 44x44 pixels)
- Keyboard navigation

**Tools:**
- Axe DevTools for accessibility auditing
- Manual testing with screen readers (TalkBack, VoiceOver)

## Deployment Architecture

### Mobile App Deployment

**Distribution:**
- Android: Google Play Store + APK for offline distribution
- iOS: Apple App Store
- Version management: Semantic versioning (MAJOR.MINOR.PATCH)
- Over-the-air updates: CodePush for non-native changes

**Release Process:**
1. Feature development in feature branches
2. Merge to develop branch for integration testing
3. Create release branch for QA testing
4. Merge to main branch for production release
5. Tag release with version number
6. Build and sign app binaries
7. Submit to app stores
8. Monitor crash reports and user feedback

### Backend Deployment

**Infrastructure:**
- Cloud provider: AWS or Azure
- Container orchestration: Kubernetes (EKS or AKS)
- Service mesh: Istio for traffic management and observability
- Load balancer: Application Load Balancer (ALB)
- Database: Amazon RDS PostgreSQL with Multi-AZ deployment
- Cache: Amazon ElastiCache Redis
- Object storage: Amazon S3 or Azure Blob Storage

**Deployment Strategy:**
- Blue-green deployment for zero-downtime releases
- Canary releases for gradual rollout (10% → 50% → 100%)
- Automated rollback on error rate threshold breach
- Database migrations with backward compatibility

**CI/CD Pipeline:**
1. Code commit triggers GitHub Actions workflow
2. Run linting and unit tests
3. Build Docker images
4. Push images to container registry (ECR/ACR)
5. Run integration tests in staging environment
6. Deploy to staging for QA approval
7. Deploy to production with canary strategy
8. Run smoke tests
9. Monitor metrics and logs

### Monitoring and Observability

**Metrics:**
- Application metrics: Request rate, error rate, latency (RED method)
- Business metrics: Active users, predictions generated, sync success rate
- Infrastructure metrics: CPU, memory, disk, network
- Custom metrics: ML model accuracy, ghost detection rate, offline usage

**Logging:**
- Structured logging with JSON format
- Log aggregation with ELK stack
- Log retention: 30 days for application logs, 90 days for audit logs
- PII redaction in logs

**Tracing:**
- Distributed tracing with Jaeger
- Trace sampling: 1% in production, 100% in staging
- Trace retention: 7 days

**Alerting:**
- Critical alerts: Page on-call engineer (error rate > 5%, latency > 5s)
- Warning alerts: Slack notification (error rate > 1%, disk > 80%)
- Info alerts: Email digest (daily summary)

### Disaster Recovery

**Backup Strategy:**
- Database: Automated daily backups with 30-day retention
- Point-in-time recovery: 5-minute granularity
- Cross-region replication for critical data
- Monthly backup restoration drills

**Recovery Objectives:**
- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 1 hour
- Failover to secondary region: Automated with health checks

## Security Architecture

### Data Encryption

**At Rest:**
- Database: Transparent Data Encryption (TDE) with AES-256
- Object storage: Server-side encryption with customer-managed keys
- Mobile app: SQLCipher for local database encryption
- Encryption key management: AWS KMS or Azure Key Vault

**In Transit:**
- TLS 1.3 for all API communications
- Certificate pinning in mobile app
- Mutual TLS (mTLS) for service-to-service communication

### Authentication and Authorization

**User Authentication:**
- Multi-factor authentication (password + OTP)
- OTP delivery via SMS (with fallback to email)
- Session management with JWT tokens
- Token expiration: Access token 15 minutes, refresh token 7 days
- Automatic logout after 15 minutes of inactivity

**Authorization:**
- Role-based access control (RBAC)
- Roles: Anganwadi Worker, Asha Worker, ANM, Supervisor, Administrator
- Permissions: Create, Read, Update, Delete on resources
- Scope: Own, Center, Block, District, All
- Policy enforcement at API gateway and service level

### Data Privacy

**PII Protection:**
- Encrypt sensitive fields (Aadhar number, phone number, address)
- Field-level encryption with separate keys
- Data masking in logs and error messages
- Access audit logging for all PII access

**Compliance:**
- GDPR-like principles for Indian context
- Data minimization: Collect only necessary data
- Purpose limitation: Use data only for stated purposes
- Data retention: Delete inactive records after 5 years
- Right to access: Users can request their data
- Right to deletion: Users can request data deletion

### Network Security

**API Security:**
- Rate limiting: 100 requests/minute per user
- API key authentication for service-to-service calls
- Input validation and sanitization
- SQL injection prevention with parameterized queries
- XSS prevention with output encoding

**Infrastructure Security:**
- VPC with private subnets for databases and services
- Security groups with least privilege access
- WAF (Web Application Firewall) for DDoS protection
- Regular security patching and updates
- Vulnerability scanning with automated tools

## Scalability Considerations

### Horizontal Scaling

**Stateless Services:**
- All backend services are stateless for easy horizontal scaling
- Session state stored in Redis
- Auto-scaling based on CPU and memory metrics
- Scale-out threshold: CPU > 70% for 5 minutes
- Scale-in threshold: CPU < 30% for 10 minutes

**Database Scaling:**
- Read replicas for read-heavy workloads
- Connection pooling to optimize database connections
- Query optimization and indexing
- Partitioning for large tables (students, measurements)
- Archival of old data to cold storage

### Caching Strategy

**Cache Layers:**
1. Mobile app: In-memory cache for frequently accessed data
2. API gateway: Response caching for static data (recipes, help content)
3. Application: Redis cache for user sessions and frequently queried data
4. Database: Query result caching

**Cache Invalidation:**
- Time-based expiration (TTL)
- Event-based invalidation on data updates
- Cache-aside pattern for application caching

### ML Model Serving

**Model Optimization:**
- Quantization for mobile models (FP32 → INT8)
- Model pruning to reduce size
- Batch inference for server-side predictions
- Model caching to avoid repeated loading

**Scaling ML Service:**
- GPU instances for model training
- CPU instances for model serving (inference)
- Model versioning and A/B testing
- Gradual rollout of new models

## Future Enhancements

1. **Advanced Analytics:**
   - Predictive analytics for disease outbreaks
   - Geospatial analysis of malnutrition hotspots
   - Trend analysis and forecasting

2. **AI Enhancements:**
   - Conversational AI chatbot for healthcare workers
   - Automated meal planning optimization
   - Image-based health assessment (skin, eyes, hair)

3. **Integration:**
   - Integration with government health systems (HMIS)
   - Integration with supply chain management systems
   - Integration with telemedicine platforms

4. **Gamification:**
   - Badges and rewards for healthcare workers
   - Leaderboards for centers with best outcomes
   - Progress tracking and goal setting

5. **Community Features:**
   - Parent engagement portal
   - Community health education content
   - Peer support groups for parents
