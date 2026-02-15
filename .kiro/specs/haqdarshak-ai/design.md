# Design Document: HaqDarshak AI

## Overview

HaqDarshak AI is a serverless, AI-powered legal violation detection system built on AWS infrastructure. The system processes user queries and document uploads through a multi-stage pipeline: OCR extraction, structured data parsing, RAG-based legal rule matching, violation detection, and complaint generation.

The architecture follows a microservices pattern with Next.js 14 for the frontend and API routes, AWS Lambda for backend processing, Amazon Bedrock (Claude Sonnet 4.5) for AI capabilities, and Supabase PostgreSQL for data persistence. The system is designed for mobile-first usage with bilingual support (Hindi/English) and sub-10-second response times.

### Key Design Principles

1. **Zero Hallucination**: All legal information comes from a curated database; AI is used only for semantic matching and text extraction
2. **Mobile-First**: Optimized for smartphone usage with responsive design and camera integration
3. **Performance**: Serverless architecture with aggressive caching and parallel processing
4. **Security**: End-to-end encryption, PII protection, and secure document storage
5. **Bilingual**: Full Hindi and English support throughout the user journey

## Architecture

### System Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        A[Next.js Frontend]
        A1[Mobile Browser]
        A2[Desktop Browser]
    end
    
    subgraph "API Layer"
        B[Next.js API Routes]
        B1[/api/upload]
        B2[/api/query]
        B3[/api/analyze]
        B4[/api/complaint]
    end
    
    subgraph "Processing Layer - AWS Lambda"
        C1[OCR Lambda]
        C2[Data Extraction Lambda]
        C3[RAG Query Lambda]
        C4[Violation Detection Lambda]
        C5[Complaint Generation Lambda]
    end
    
    subgraph "AI Services"
        D1[Amazon Bedrock - Claude Sonnet 4.5]
        D2[OCR Engine - Tesseract/Google Vision]
    end
    
    subgraph "Data Layer"
        E1[AWS S3 - Document Storage]
        E2[Supabase PostgreSQL]
        E3[Legal Rules JSON Database]
        E4[Redis Cache]
    end
    
    A --> B
    A1 --> A
    A2 --> A
    B --> B1
    B --> B2
    B --> B3
    B --> B4
    
    B1 --> C1
    B2 --> C2
    B3 --> C3
    B4 --> C5
    
    C1 --> D2
    C2 --> D1
    C3 --> D1
    C3 --> E3
    C4 --> D1
    C5 --> D1
    
    C1 --> E1
    C2 --> E2
    C3 --> E4
    C4 --> E2
    C5 --> E2
```

### Data Flow

1. **Document Upload Flow**:
   - User uploads document → Next.js API → S3 storage → OCR Lambda
   - OCR Lambda extracts text → Data Extraction Lambda structures data
   - Structured data → RAG Query Lambda → Violation Detection Lambda
   - Results returned to frontend with complaint options

2. **Text Query Flow**:
   - User submits query → Next.js API → Data Extraction Lambda
   - Extracted intent → RAG Query Lambda → Violation Detection Lambda
   - Results returned with relevant legal information

3. **Complaint Generation Flow**:
   - User requests complaint → Complaint Generation Lambda
   - Template selection → Data population → Format conversion (PDF/WhatsApp/Email)
   - Downloadable complaint returned to user

## Components and Interfaces

### 1. Frontend Components (Next.js 14 + React + TypeScript)

#### 1.1 Landing Page Component
```typescript
interface LandingPageProps {
  onUploadClick: () => void;
  onQueryClick: () => void;
  language: 'en' | 'hi';
}

// Displays hero section, use case cards, and CTA buttons
```

#### 1.2 Upload Component
```typescript
interface UploadComponentProps {
  onFileSelect: (file: File) => void;
  acceptedFormats: string[]; // ['image/jpeg', 'image/png', 'application/pdf']
  maxSizeMB: number; // 10
  language: 'en' | 'hi';
}

interface UploadResponse {
  documentId: string;
  s3Url: string;
  status: 'uploaded' | 'processing' | 'completed' | 'failed';
}
```

#### 1.3 Query Input Component
```typescript
interface QueryInputProps {
  onSubmit: (query: string) => void;
  language: 'en' | 'hi';
  maxLength: number; // 500
}
```

#### 1.4 Results Display Component
```typescript
interface ResultsDisplayProps {
  verdict: VerdictData;
  extractedData: StructuredData;
  legalCitations: LegalCitation[];
  actionOptions: ActionOption[];
  language: 'en' | 'hi';
}

interface VerdictData {
  status: 'violation_detected' | 'legal' | 'insufficient_info';
  title: string;
  explanation: string;
  confidence: number; // 0-100
  disclaimer: string;
}

interface StructuredData {
  amount: number | null;
  vendor: string | null;
  chargeType: 'mrp' | 'service_charge' | 'challan' | 'other';
  date: string | null;
  products: Product[];
  editable: boolean;
}

interface Product {
  name: string;
  price: number;
  mrp: number | null;
}

interface LegalCitation {
  law: string;
  section: string;
  description: string;
  relevance: number;
}

interface ActionOption {
  type: 'whatsapp' | 'email' | 'pdf' | 'share';
  label: string;
  action: () => void;
}
```

#### 1.5 Complaint Generator Component
```typescript
interface ComplaintGeneratorProps {
  violationData: ViolationData;
  language: 'en' | 'hi';
  onGenerate: (format: ComplaintFormat) => void;
}

interface ViolationData {
  documentId: string;
  structuredData: StructuredData;
  verdict: VerdictData;
  legalCitations: LegalCitation[];
}

type ComplaintFormat = 'whatsapp' | 'email' | 'pdf';

interface ComplaintOutput {
  format: ComplaintFormat;
  content: string;
  downloadUrl?: string; // For PDF
  recipientInfo?: RecipientInfo; // For email
}

interface RecipientInfo {
  authority: string;
  email: string;
  subject: string;
}
```

### 2. API Routes (Next.js API Routes)

#### 2.1 Document Upload API
```typescript
// POST /api/upload
interface UploadRequest {
  file: File;
  language: 'en' | 'hi';
}

interface UploadAPIResponse {
  success: boolean;
  documentId: string;
  s3Url: string;
  message: string;
}
```

#### 2.2 Text Query API
```typescript
// POST /api/query
interface QueryRequest {
  query: string;
  language: 'en' | 'hi';
}

interface QueryAPIResponse {
  success: boolean;
  queryId: string;
  extractedIntent: string;
  message: string;
}
```

#### 2.3 Analysis API
```typescript
// POST /api/analyze
interface AnalyzeRequest {
  documentId?: string;
  queryId?: string;
  manualData?: StructuredData; // For manual entry fallback
}

interface AnalyzeAPIResponse {
  success: boolean;
  verdict: VerdictData;
  structuredData: StructuredData;
  legalCitations: LegalCitation[];
  processingTimeMs: number;
}
```

#### 2.4 Complaint Generation API
```typescript
// POST /api/complaint
interface ComplaintRequest {
  violationData: ViolationData;
  format: ComplaintFormat;
  userContact?: UserContact;
}

interface UserContact {
  name: string;
  email: string;
  phone: string;
}

interface ComplaintAPIResponse {
  success: boolean;
  complaint: ComplaintOutput;
  generatedAt: string;
}
```

### 3. AWS Lambda Functions

#### 3.1 OCR Lambda Function
```typescript
interface OCRLambdaInput {
  s3Bucket: string;
  s3Key: string;
  documentId: string;
}

interface OCRLambdaOutput {
  documentId: string;
  extractedText: string;
  confidence: number;
  language: 'en' | 'hi' | 'mixed';
  processingTimeMs: number;
}

// Uses Tesseract.js or Google Vision API
// Handles image preprocessing (deskew, contrast enhancement)
// Returns raw text with confidence scores
```

#### 3.2 Data Extraction Lambda Function
```typescript
interface DataExtractionInput {
  text: string;
  source: 'ocr' | 'query';
  language: 'en' | 'hi';
}

interface DataExtractionOutput {
  structuredData: StructuredData;
  confidence: Record<string, number>; // Field-level confidence
  extractionMethod: 'bedrock' | 'regex' | 'hybrid';
}

// Uses Amazon Bedrock for intelligent extraction
// Falls back to regex patterns for structured fields
// Validates extracted data against expected formats
```

#### 3.3 RAG Query Lambda Function
```typescript
interface RAGQueryInput {
  structuredData: StructuredData;
  query?: string;
  language: 'en' | 'hi';
}

interface RAGQueryOutput {
  matchedRules: LegalRule[];
  relevanceScores: number[];
  queryEmbedding: number[];
  cacheHit: boolean;
}

interface LegalRule {
  rule_id: string;
  category: 'mrp' | 'service_charge' | 'challan' | 'other';
  law: string;
  section: string;
  description: string;
  violation_keywords: string[];
  penalty: string;
  authority: string;
  complaint_template_id: string;
}

// Generates embeddings using Bedrock
// Queries vector database or JSON with semantic search
// Returns top-k relevant rules with scores
```

#### 3.4 Violation Detection Lambda Function
```typescript
interface ViolationDetectionInput {
  structuredData: StructuredData;
  matchedRules: LegalRule[];
  language: 'en' | 'hi';
}

interface ViolationDetectionOutput {
  verdict: VerdictData;
  violationType: string | null;
  applicableRules: LegalRule[];
  reasoning: string;
}

// Applies rule-based logic for violation detection
// Uses Bedrock for natural language explanation generation
// Ensures deterministic violation detection (no hallucination)
```

#### 3.5 Complaint Generation Lambda Function
```typescript
interface ComplaintGenerationInput {
  violationData: ViolationData;
  format: ComplaintFormat;
  userContact?: UserContact;
  language: 'en' | 'hi';
}

interface ComplaintGenerationOutput {
  complaint: ComplaintOutput;
  templateUsed: string;
  generatedAt: string;
}

// Loads complaint template from database
// Populates template with violation data
// Formats for target channel (WhatsApp/Email/PDF)
// Generates PDF using library like PDFKit
```

### 4. Data Models

#### 4.1 Database Schema (Supabase PostgreSQL)

```sql
-- Documents table
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_session_id VARCHAR(255),
  s3_url TEXT NOT NULL,
  file_type VARCHAR(10) NOT NULL,
  file_size_bytes INTEGER,
  uploaded_at TIMESTAMP DEFAULT NOW(),
  ocr_status VARCHAR(20) DEFAULT 'pending',
  ocr_text TEXT,
  ocr_confidence DECIMAL(5,2),
  deleted_at TIMESTAMP NULL
);

-- Analyses table
CREATE TABLE analyses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents(id),
  query_text TEXT,
  structured_data JSONB NOT NULL,
  verdict JSONB NOT NULL,
  legal_citations JSONB NOT NULL,
  processing_time_ms INTEGER,
  language VARCHAR(2) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Complaints table
CREATE TABLE complaints (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  analysis_id UUID REFERENCES analyses(id),
  format VARCHAR(20) NOT NULL,
  content TEXT NOT NULL,
  pdf_url TEXT,
  user_contact JSONB,
  generated_at TIMESTAMP DEFAULT NOW()
);

-- Legal rules table (can also be JSON file)
CREATE TABLE legal_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id VARCHAR(50) UNIQUE NOT NULL,
  category VARCHAR(50) NOT NULL,
  law VARCHAR(255) NOT NULL,
  section VARCHAR(100) NOT NULL,
  description TEXT NOT NULL,
  description_hi TEXT,
  violation_keywords TEXT[],
  penalty TEXT,
  authority VARCHAR(255),
  complaint_template_id VARCHAR(50),
  version INTEGER DEFAULT 1,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Complaint templates table
CREATE TABLE complaint_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id VARCHAR(50) UNIQUE NOT NULL,
  category VARCHAR(50) NOT NULL,
  template_text TEXT NOT NULL,
  template_text_hi TEXT,
  placeholders TEXT[],
  created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_documents_session ON documents(user_session_id);
CREATE INDEX idx_documents_uploaded ON documents(uploaded_at);
CREATE INDEX idx_analyses_document ON analyses(document_id);
CREATE INDEX idx_analyses_created ON analyses(created_at);
CREATE INDEX idx_legal_rules_category ON legal_rules(category);
CREATE INDEX idx_legal_rules_keywords ON legal_rules USING GIN(violation_keywords);
```

#### 4.2 Legal Rules JSON Structure

```json
{
  "rules": [
    {
      "rule_id": "LM_MRP_001",
      "category": "mrp",
      "law": "Legal Metrology Act, 2009",
      "section": "18(1)",
      "description": "No person shall sell or distribute any packaged commodity at a price exceeding the maximum retail price declared on the package",
      "description_hi": "कोई भी व्यक्ति पैकेज पर घोषित अधिकतम खुदरा मूल्य से अधिक कीमत पर कोई पैकेज्ड वस्तु नहीं बेच सकता",
      "violation_keywords": ["MRP", "overcharge", "maximum retail price", "price above MRP"],
      "penalty": "Fine up to ₹25,000 or imprisonment up to 1 year",
      "authority": "Legal Metrology Department",
      "complaint_template_id": "TMPL_MRP_001"
    },
    {
      "rule_id": "CCPA_SC_001",
      "category": "service_charge",
      "law": "CCPA Guidelines on Service Charges",
      "section": "2022 Guidelines",
      "description": "Service charges are voluntary and cannot be automatically added to bills. Customers have the right to refuse payment",
      "description_hi": "सेवा शुल्क स्वैच्छिक हैं और स्वचालित रूप से बिल में नहीं जोड़े जा सकते। ग्राहकों को भुगतान से इनकार करने का अधिकार है",
      "violation_keywords": ["service charge", "mandatory", "automatic", "compulsory"],
      "penalty": "Fine up to ₹10 lakh",
      "authority": "Central Consumer Protection Authority (CCPA)",
      "complaint_template_id": "TMPL_SC_001"
    },
    {
      "rule_id": "MVA_CHALLAN_001",
      "category": "challan",
      "law": "Motor Vehicles Act, 1988",
      "section": "177-194",
      "description": "Traffic fines must not exceed the amounts specified in the Motor Vehicles Act penalty schedule",
      "description_hi": "यातायात जुर्माना मोटर वाहन अधिनियम दंड अनुसूची में निर्दिष्ट राशि से अधिक नहीं होना चाहिए",
      "violation_keywords": ["challan", "traffic fine", "penalty", "overcharged fine"],
      "penalty": "Varies by violation type",
      "authority": "Regional Transport Office (RTO) / Traffic Police",
      "complaint_template_id": "TMPL_CHALLAN_001"
    }
  ]
}
```

### 5. External Service Integrations

#### 5.1 Amazon Bedrock Integration
```typescript
interface BedrockConfig {
  region: 'ap-south-1'; // Mumbai region
  model: 'anthropic.claude-3-5-sonnet-20241022-v2:0';
  maxTokens: 4096;
  temperature: 0.1; // Low temperature for consistency
}

interface BedrockRequest {
  prompt: string;
  systemPrompt?: string;
  maxTokens: number;
  temperature: number;
}

interface BedrockResponse {
  completion: string;
  stopReason: string;
  usage: {
    inputTokens: number;
    outputTokens: number;
  };
}

// Used for:
// - Structured data extraction from OCR text
// - Semantic matching of queries to legal rules
// - Natural language explanation generation
// - Complaint text generation
```

#### 5.2 OCR Engine Integration
```typescript
interface OCRConfig {
  engine: 'tesseract' | 'google-vision';
  languages: ['eng', 'hin'];
  preprocessingSteps: ['deskew', 'contrast', 'denoise'];
}

// Tesseract.js for cost-effective OCR
// Google Vision API as fallback for complex documents
// Preprocessing pipeline for image quality enhancement
```

#### 5.3 AWS S3 Integration
```typescript
interface S3Config {
  bucket: string;
  region: 'ap-south-1';
  encryption: 'AES256';
  lifecyclePolicy: {
    deleteAfterDays: 30; // Auto-delete documents after 30 days
  };
}

interface S3UploadParams {
  key: string;
  body: Buffer;
  contentType: string;
  metadata: Record<string, string>;
}
```

## Data Models

### Frontend State Management

```typescript
// Global app state (using React Context or Zustand)
interface AppState {
  language: 'en' | 'hi';
  currentDocument: DocumentState | null;
  currentAnalysis: AnalysisState | null;
  loading: boolean;
  error: ErrorState | null;
}

interface DocumentState {
  id: string;
  file: File;
  uploadProgress: number;
  ocrStatus: 'pending' | 'processing' | 'completed' | 'failed';
  extractedText: string | null;
}

interface AnalysisState {
  id: string;
  verdict: VerdictData;
  structuredData: StructuredData;
  legalCitations: LegalCitation[];
  complaintGenerated: boolean;
}

interface ErrorState {
  code: string;
  message: string;
  retryable: boolean;
}
```

### Caching Strategy

```typescript
interface CacheConfig {
  redis: {
    host: string;
    port: number;
    ttl: {
      legalRules: 86400; // 24 hours
      ocrResults: 3600; // 1 hour
      analysisResults: 1800; // 30 minutes
    };
  };
}

// Cache keys
const CACHE_KEYS = {
  legalRule: (ruleId: string) => `legal:rule:${ruleId}`,
  ocrResult: (documentId: string) => `ocr:${documentId}`,
  analysisResult: (analysisId: string) => `analysis:${analysisId}`,
  queryEmbedding: (queryHash: string) => `embedding:${queryHash}`,
};
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties could be consolidated:

1. **Language Handling**: Properties 1.1 and 1.2 (English/Hindi query processing) can be combined into a single property about language consistency
2. **File Format Acceptance**: Properties 2.1, 2.2, 2.3 (JPG/PNG/PDF) are specific examples, not universal properties
3. **Data Extraction**: Properties 4.1-4.5 (extracting amount, vendor, charge type, date, products) can be combined into a comprehensive extraction completeness property
4. **Complaint Format Generation**: Properties 11.1, 11.2, 11.3 (WhatsApp/Email/PDF) can be combined into a multi-format generation property
5. **Authority Mapping**: Properties 7.5, 8.5, 9.5 (authority identification) can be combined into a single property about correct authority mapping

### Core Properties

#### Property 1: Language Consistency
*For any* text query or document in a specific language (English or Hindi), the system response (verdict, complaint, UI text) should be in the same language as the input or user's selected preference.

**Validates: Requirements 1.1, 1.2, 10.6, 17.2, 17.3**

#### Property 2: Query Length Validation
*For any* text query, if the length is 500 characters or less, it should be accepted; if greater than 500 characters, it should be rejected with an error message.

**Validates: Requirements 1.4**

#### Property 3: File Type Validation
*For any* uploaded file, if the file format is in the set of unsupported formats (not JPG, PNG, or PDF), the system should reject the upload and display an error message.

**Validates: Requirements 2.5**

#### Property 4: Document Encryption at Rest
*For any* document uploaded to S3, the S3 object metadata should indicate encryption is enabled (AES256 or KMS).

**Validates: Requirements 2.6, 14.1**

#### Property 5: OCR Text Extraction Completeness
*For any* successfully processed document, the OCR engine should return non-empty extracted text or trigger the manual entry fallback.

**Validates: Requirements 3.1, 3.5**

#### Property 6: Structured Data Extraction Completeness
*For any* raw text containing transaction information, the system should extract and populate all applicable fields (amount, vendor, charge type, date, products) in the structured data JSON output.

**Validates: Requirements 4.1, 4.2, 4.3, 4.4, 4.5, 4.7**

#### Property 7: Low Confidence Field Flagging
*For any* extracted data field with confidence below 70%, the system should mark that field as requiring user verification in the structured data output.

**Validates: Requirements 4.6**

#### Property 8: RAG Zero Hallucination
*For any* legal rule returned by the RAG system, the rule_id must exist in the curated Legal_Database, ensuring no hallucinated legal information is generated.

**Validates: Requirements 5.5**

#### Property 9: Rule Relevance Ranking
*For any* query that matches multiple legal rules, the returned rules should be ordered by descending relevance score.

**Validates: Requirements 5.6**

#### Property 10: Verdict Citation Completeness
*For any* generated verdict, the verdict object should contain at least one legal citation with law name and section number.

**Validates: Requirements 6.5**

#### Property 11: Verdict Explanation Presence
*For any* generated verdict, the verdict object should contain a non-empty explanation field describing why the charge is legal or illegal.

**Validates: Requirements 6.6**

#### Property 12: Legal Disclaimer Presence
*For any* verdict displayed to the user, the UI should include the legal disclaimer text "This is for informational purposes only and does not constitute legal advice".

**Validates: Requirements 6.7**

#### Property 13: MRP Overcharge Detection
*For any* product where charged price exceeds MRP, the system should flag this as a violation and cite Legal Metrology Act Section 18(1).

**Validates: Requirements 7.1, 7.2**

#### Property 14: Service Charge Violation Detection
*For any* bill containing a service charge marked as mandatory or automatically added, the system should flag this as a violation and cite CCPA guidelines.

**Validates: Requirements 8.1, 8.3**

#### Property 15: Challan Amount Validation
*For any* traffic challan where the fine amount exceeds the Motor Vehicles Act penalty schedule for that violation type, the system should flag this as an invalid challan.

**Validates: Requirements 9.1, 9.2, 9.3**

#### Property 16: Complaint Data Completeness
*For any* generated complaint, all extracted transaction details (amount, vendor, date, charge type) should appear in the complaint text.

**Validates: Requirements 10.2**

#### Property 17: Complaint Legal Citation Inclusion
*For any* generated complaint, the complaint text should include the relevant law name and section number from the violation verdict.

**Validates: Requirements 10.3**

#### Property 18: Complaint Authority Addressing
*For any* violation type (MRP, service charge, challan), the generated complaint should be addressed to the correct authority as defined in the legal rule (Legal Metrology Department, CCPA, RTO/Traffic Police).

**Validates: Requirements 10.4, 7.5, 8.5, 9.5**

#### Property 19: Multi-Format Complaint Generation
*For any* violation with a generated complaint, the system should be able to produce the complaint in all three formats (WhatsApp message, email draft, PDF) upon request.

**Validates: Requirements 11.1, 11.2, 11.3**

#### Property 20: WhatsApp Message Length Constraint
*For any* WhatsApp-formatted complaint, the message length should not exceed WhatsApp's character limit (currently 65,536 characters) while preserving all key information (violation type, amount, law citation, authority).

**Validates: Requirements 11.4**

#### Property 21: PDF Complaint Validity
*For any* generated PDF complaint, the file should be a valid PDF that can be opened by standard PDF readers and contains properly formatted text.

**Validates: Requirements 11.6**

#### Property 22: Contact Information Privacy
*For any* user session where explicit consent for storing contact information has not been given, the system should not persist user contact details (name, email, phone) to the database.

**Validates: Requirements 14.3**

#### Property 23: Data Anonymization
*For any* extracted data used for system analytics or improvement, all personally identifiable information (names, phone numbers, email addresses, specific addresses) should be removed or replaced with placeholders.

**Validates: Requirements 14.5**

#### Property 24: Legal Rule Schema Validation
*For any* legal rule in the Legal_Database, the rule JSON should validate against the standardized schema and contain all required fields (rule_id, category, law, section, description, violation_keywords, penalty, authority).

**Validates: Requirements 15.1, 15.2, 15.3**

#### Property 25: Legal Rule Versioning
*For any* update to an existing legal rule, the system should create a new version with a timestamp while preserving the previous version in the version history.

**Validates: Requirements 15.4, 15.5**

#### Property 26: Error Logging Completeness
*For any* error that occurs during processing, the system should log an error entry containing at minimum: timestamp, error type, error message, user session ID, and processing stage.

**Validates: Requirements 16.6**

#### Property 27: Language Toggle Presence
*For any* page in the application, the UI should include a language toggle control allowing users to switch between Hindi and English.

**Validates: Requirements 17.1**

#### Property 28: Legal Citation Translation Format
*For any* legal citation displayed in Hindi, the translated text should include the English Act name in parentheses (e.g., "कानूनी मेट्रोलॉजी अधिनियम (Legal Metrology Act)").

**Validates: Requirements 17.4**

#### Property 29: Language Preference Persistence
*For any* user who selects a language preference, that preference should be stored (in localStorage or cookies) and applied automatically on subsequent visits.

**Validates: Requirements 17.5**

#### Property 30: Results Page Structured Data Editability
*For any* structured data field displayed on the results page, the UI should provide an edit control allowing users to correct extraction errors.

**Validates: Requirements 19.2**

#### Property 31: Violation Action Button Display
*For any* analysis result where a violation is detected, the results page should display action buttons for complaint generation (WhatsApp, Email, PDF).

**Validates: Requirements 19.4**

#### Property 32: Legal Charge Educational Content Display
*For any* analysis result where no violation is detected, the results page should display educational information about the relevant consumer protection law.

**Validates: Requirements 19.5**

### Edge Case Properties

These properties specifically test boundary conditions and error scenarios:

#### Edge Case 1: File Size Boundary
*For any* uploaded file with size exactly at or above 10MB, the system should reject the upload; for files below 10MB, the system should accept the upload.

**Validates: Requirements 2.4**

#### Edge Case 2: Mixed Language Query Detection
*For any* query containing both Hindi and English text, the system should detect the language with the higher character count as the primary language and respond in that language.

**Validates: Requirements 1.3**

#### Edge Case 3: Insufficient Query Information
*For any* query that lacks required information to perform analysis (e.g., no amount mentioned, no charge type identifiable), the system should prompt the user for the missing details rather than proceeding with incomplete data.

**Validates: Requirements 1.5**

## Error Handling

### Error Categories and Handling Strategies

#### 1. Upload Errors
- **Invalid File Format**: Return 400 error with message listing accepted formats
- **File Too Large**: Return 413 error with message indicating 10MB limit
- **S3 Upload Failure**: Retry up to 3 times with exponential backoff, then return 500 error

#### 2. OCR Errors
- **OCR Engine Failure**: Display manual text entry form as fallback
- **Low Confidence Extraction (<85%)**: Flag low-confidence fields and allow user editing
- **Unreadable Document**: Provide image quality improvement suggestions (better lighting, focus, angle)

#### 3. AI Service Errors
- **Bedrock Service Unavailable**: Display retry button with estimated wait time, cache last successful request
- **Bedrock Timeout**: Retry once, then fall back to rule-based matching without AI explanation
- **Rate Limiting**: Queue request and display position in queue

#### 4. Data Processing Errors
- **Structured Data Extraction Failure**: Provide manual entry form with guided fields
- **No Legal Rules Matched**: Display general consumer rights information and suggest consulting legal expert
- **Ambiguous Violation Detection**: Present multiple possible interpretations and ask user to clarify

#### 5. Network Errors
- **Connection Lost During Upload**: Save progress to localStorage, resume when connection restored
- **API Timeout**: Retry with exponential backoff (1s, 2s, 4s), max 3 attempts
- **DNS Resolution Failure**: Display offline mode message with cached legal information

#### 6. Database Errors
- **Supabase Connection Failure**: Retry with connection pool, fall back to read-only cached data
- **Query Timeout**: Optimize query or return partial results with warning
- **Data Integrity Violation**: Log error, rollback transaction, return 500 error

### Error Response Format

```typescript
interface ErrorResponse {
  success: false;
  error: {
    code: string; // e.g., "UPLOAD_FILE_TOO_LARGE"
    message: string; // User-friendly message in selected language
    details?: string; // Technical details for debugging
    retryable: boolean;
    retryAfterMs?: number;
    fallbackAction?: {
      type: 'manual_entry' | 'retry' | 'contact_support';
      label: string;
    };
  };
}
```

### Error Logging

All errors should be logged with structured data:

```typescript
interface ErrorLog {
  timestamp: string;
  errorCode: string;
  errorMessage: string;
  userSessionId: string;
  documentId?: string;
  processingStage: 'upload' | 'ocr' | 'extraction' | 'rag' | 'violation_detection' | 'complaint_generation';
  stackTrace?: string;
  requestMetadata: {
    userAgent: string;
    language: string;
    fileType?: string;
    fileSize?: number;
  };
}
```

## Testing Strategy

### Dual Testing Approach

The testing strategy employs both unit tests and property-based tests to ensure comprehensive coverage:

- **Unit Tests**: Verify specific examples, edge cases, and error conditions
- **Property Tests**: Verify universal properties across all inputs using randomized test data

Both approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide input space.

### Property-Based Testing Configuration

**Library Selection**:
- **JavaScript/TypeScript**: Use `fast-check` library for property-based testing
- **Minimum Iterations**: 100 runs per property test (due to randomization)
- **Test Tagging**: Each property test must include a comment referencing the design property

**Tag Format**:
```typescript
// Feature: haqdarshak-ai, Property 1: Language Consistency
test('responses match input language', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        query: fc.string(),
        language: fc.constantFrom('en', 'hi')
      }),
      async ({ query, language }) => {
        const response = await processQuery(query, language);
        expect(response.language).toBe(language);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing Focus Areas

Unit tests should focus on:

1. **Specific Examples**:
   - Valid JPG/PNG/PDF file uploads
   - MRP overcharge with specific amounts
   - Service charge violation with sample bill
   - Traffic challan with known violation types

2. **Edge Cases**:
   - File exactly at 10MB size limit
   - Query exactly at 500 character limit
   - OCR confidence exactly at 70% threshold
   - Empty or whitespace-only queries

3. **Error Conditions**:
   - Unsupported file formats
   - Network timeouts
   - Bedrock service unavailability
   - Invalid JSON in legal rules database

4. **Integration Points**:
   - S3 upload and retrieval
   - Bedrock API calls
   - Database queries and updates
   - PDF generation

### Property-Based Testing Focus Areas

Property tests should focus on:

1. **Universal Invariants**:
   - Language consistency across all queries
   - Encryption for all uploaded documents
   - Citation presence in all verdicts
   - Schema validity for all legal rules

2. **Data Transformation Properties**:
   - OCR extraction produces text for all valid images
   - Structured data extraction produces valid JSON for all text inputs
   - Complaint generation produces valid output for all violation inputs

3. **Boundary Behavior**:
   - File size validation for all file sizes
   - Query length validation for all query lengths
   - Confidence threshold behavior for all confidence values

4. **Relationship Properties**:
   - MRP violations detected when price > MRP for all products
   - Correct authority mapping for all violation types
   - Multi-format generation produces consistent content across formats

### Test Data Generation

**For Property Tests**:
```typescript
// Generate random bills with varying structures
const billGenerator = fc.record({
  vendor: fc.string({ minLength: 3, maxLength: 50 }),
  amount: fc.float({ min: 1, max: 100000 }),
  items: fc.array(fc.record({
    name: fc.string(),
    price: fc.float({ min: 1, max: 10000 }),
    mrp: fc.option(fc.float({ min: 1, max: 10000 }))
  })),
  date: fc.date(),
  chargeType: fc.constantFrom('mrp', 'service_charge', 'challan', 'other')
});

// Generate random queries in both languages
const queryGenerator = fc.record({
  text: fc.oneof(
    fc.string({ minLength: 10, maxLength: 500 }), // English
    fc.string({ minLength: 10, maxLength: 500 })  // Hindi (would use Hindi character set)
  ),
  language: fc.constantFrom('en', 'hi')
});

// Generate random legal rules
const legalRuleGenerator = fc.record({
  rule_id: fc.string({ minLength: 5, maxLength: 20 }),
  category: fc.constantFrom('mrp', 'service_charge', 'challan'),
  law: fc.string(),
  section: fc.string(),
  description: fc.string({ minLength: 20 }),
  violation_keywords: fc.array(fc.string(), { minLength: 1, maxLength: 10 }),
  penalty: fc.string(),
  authority: fc.string()
});
```

### Test Coverage Goals

- **Unit Test Coverage**: Minimum 80% code coverage
- **Property Test Coverage**: All 32 core properties + 3 edge case properties implemented
- **Integration Test Coverage**: All API endpoints and Lambda functions
- **E2E Test Coverage**: All three primary use cases (MRP, service charge, challan)

### Continuous Testing

- **Pre-commit**: Run unit tests and linting
- **CI Pipeline**: Run full test suite (unit + property + integration)
- **Nightly**: Run extended property tests with 1000 iterations
- **Pre-deployment**: Run E2E tests against staging environment

### Performance Testing

While not part of unit/property testing, performance requirements should be validated through:

- **Load Testing**: Simulate 1000 concurrent users
- **Stress Testing**: Test system behavior under extreme load
- **Latency Testing**: Verify <10 second end-to-end response time
- **OCR Performance**: Verify 85%+ accuracy on test document set


## API Specifications

### REST API Endpoints

#### 1. POST /api/upload

Upload a document for analysis.

**Request**:
```typescript
Content-Type: multipart/form-data

{
  file: File, // JPG, PNG, or PDF
  language: 'en' | 'hi'
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "documentId": "uuid-v4",
  "s3Url": "https://s3.amazonaws.com/bucket/key",
  "message": "Document uploaded successfully"
}
```

**Error Responses**:
- 400: Invalid file format or missing file
- 413: File size exceeds 10MB
- 500: S3 upload failure

#### 2. POST /api/query

Submit a text query for analysis.

**Request**:
```json
{
  "query": "string (max 500 chars)",
  "language": "en" | "hi"
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "queryId": "uuid-v4",
  "extractedIntent": "string",
  "message": "Query processed successfully"
}
```

**Error Responses**:
- 400: Query too long or empty
- 500: Processing failure

#### 3. POST /api/analyze

Analyze uploaded document or query.

**Request**:
```json
{
  "documentId": "uuid-v4", // Optional, for document analysis
  "queryId": "uuid-v4", // Optional, for query analysis
  "manualData": { // Optional, for manual entry fallback
    "amount": 150.50,
    "vendor": "ABC Store",
    "chargeType": "mrp",
    "date": "2024-01-15",
    "products": [
      {
        "name": "Product A",
        "price": 150.50,
        "mrp": 140.00
      }
    ]
  }
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "verdict": {
    "status": "violation_detected" | "legal" | "insufficient_info",
    "title": "Illegal Charge Detected",
    "explanation": "The charged price of ₹150.50 exceeds the MRP of ₹140.00...",
    "confidence": 95,
    "disclaimer": "This is for informational purposes only..."
  },
  "structuredData": {
    "amount": 150.50,
    "vendor": "ABC Store",
    "chargeType": "mrp",
    "date": "2024-01-15",
    "products": [
      {
        "name": "Product A",
        "price": 150.50,
        "mrp": 140.00
      }
    ],
    "editable": true
  },
  "legalCitations": [
    {
      "law": "Legal Metrology Act, 2009",
      "section": "18(1)",
      "description": "No person shall sell or distribute...",
      "relevance": 0.98
    }
  ],
  "processingTimeMs": 8500
}
```

**Error Responses**:
- 400: Invalid documentId or queryId
- 404: Document or query not found
- 500: Analysis failure

#### 4. POST /api/complaint

Generate a complaint document.

**Request**:
```json
{
  "violationData": {
    "documentId": "uuid-v4",
    "structuredData": { /* ... */ },
    "verdict": { /* ... */ },
    "legalCitations": [ /* ... */ ]
  },
  "format": "whatsapp" | "email" | "pdf",
  "userContact": { // Optional
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+91-9876543210"
  }
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "complaint": {
    "format": "pdf",
    "content": "To: Legal Metrology Department\n\nSubject: Complaint regarding MRP violation...",
    "downloadUrl": "https://s3.amazonaws.com/bucket/complaint-uuid.pdf",
    "recipientInfo": {
      "authority": "Legal Metrology Department",
      "email": "legalmetrology@example.gov.in",
      "subject": "MRP Violation Complaint - ABC Store"
    }
  },
  "generatedAt": "2024-01-15T10:30:00Z"
}
```

**Error Responses**:
- 400: Invalid violation data or format
- 500: Complaint generation failure

#### 5. GET /api/document/:documentId

Retrieve document details and analysis status.

**Response** (200 OK):
```json
{
  "success": true,
  "document": {
    "id": "uuid-v4",
    "s3Url": "https://...",
    "fileType": "image/jpeg",
    "uploadedAt": "2024-01-15T10:00:00Z",
    "ocrStatus": "completed",
    "ocrText": "Extracted text...",
    "ocrConfidence": 92.5
  }
}
```

#### 6. DELETE /api/document/:documentId

Delete a document and associated data.

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Document marked for deletion"
}
```

### Lambda Function Interfaces

#### OCR Lambda

**Input Event**:
```json
{
  "s3Bucket": "haqdarshak-documents",
  "s3Key": "documents/uuid-v4.jpg",
  "documentId": "uuid-v4"
}
```

**Output**:
```json
{
  "documentId": "uuid-v4",
  "extractedText": "Bill\nABC Store\nTotal: ₹150.50...",
  "confidence": 92.5,
  "language": "en",
  "processingTimeMs": 3200
}
```

#### Data Extraction Lambda

**Input Event**:
```json
{
  "text": "Bill\nABC Store\nTotal: ₹150.50...",
  "source": "ocr",
  "language": "en"
}
```

**Output**:
```json
{
  "structuredData": {
    "amount": 150.50,
    "vendor": "ABC Store",
    "chargeType": "mrp",
    "date": "2024-01-15",
    "products": []
  },
  "confidence": {
    "amount": 0.95,
    "vendor": 0.88,
    "chargeType": 0.92,
    "date": 0.85
  },
  "extractionMethod": "bedrock"
}
```

#### RAG Query Lambda

**Input Event**:
```json
{
  "structuredData": {
    "amount": 150.50,
    "chargeType": "mrp"
  },
  "query": "charged above MRP",
  "language": "en"
}
```

**Output**:
```json
{
  "matchedRules": [
    {
      "rule_id": "LM_MRP_001",
      "category": "mrp",
      "law": "Legal Metrology Act, 2009",
      "section": "18(1)",
      "description": "No person shall sell...",
      "violation_keywords": ["MRP", "overcharge"],
      "penalty": "Fine up to ₹25,000",
      "authority": "Legal Metrology Department",
      "complaint_template_id": "TMPL_MRP_001"
    }
  ],
  "relevanceScores": [0.98],
  "queryEmbedding": [0.123, 0.456, ...],
  "cacheHit": false
}
```

## Security Considerations

### Authentication and Authorization

**Phase 1 (MVP)**: No authentication required
- Session-based tracking using anonymous session IDs
- Rate limiting by IP address
- No user accounts or login

**Phase 2 (Future)**: Optional user accounts
- OAuth 2.0 integration (Google, Phone OTP)
- JWT-based authentication
- Role-based access control for admin functions

### Data Protection

#### 1. Encryption

**At Rest**:
- S3 documents: AES-256 encryption
- Database: Supabase encryption at rest
- Secrets: AWS Secrets Manager

**In Transit**:
- TLS 1.3 for all API communications
- HTTPS only (no HTTP)
- Certificate pinning for mobile apps (future)

#### 2. PII Handling

**Minimization**:
- Collect only essential data
- No user contact info stored without consent
- Anonymous session IDs instead of user tracking

**Anonymization**:
- Remove names, phone numbers, emails from analytics data
- Hash vendor names for aggregation
- Redact specific addresses

**Retention**:
- Documents auto-deleted after 30 days
- Analysis results retained for 90 days
- Complaint history retained for 1 year (if user consents)

#### 3. Access Control

**S3 Bucket Policies**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::haqdarshak-documents/*"
    }
  ]
}
```

**Lambda IAM Roles**:
- Principle of least privilege
- Separate roles for each Lambda function
- No wildcard permissions

**Database Access**:
- Row-level security in Supabase
- Connection pooling with limited connections
- Read-only replicas for analytics

### Input Validation

#### File Upload Validation

```typescript
function validateUpload(file: File): ValidationResult {
  // File type check
  const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
  if (!allowedTypes.includes(file.type)) {
    return { valid: false, error: 'INVALID_FILE_TYPE' };
  }
  
  // File size check
  const maxSizeBytes = 10 * 1024 * 1024; // 10MB
  if (file.size > maxSizeBytes) {
    return { valid: false, error: 'FILE_TOO_LARGE' };
  }
  
  // File name sanitization
  const sanitizedName = file.name.replace(/[^a-zA-Z0-9.-]/g, '_');
  
  // Magic number validation (check actual file content)
  const magicNumbers = {
    'image/jpeg': [0xFF, 0xD8, 0xFF],
    'image/png': [0x89, 0x50, 0x4E, 0x47],
    'application/pdf': [0x25, 0x50, 0x44, 0x46]
  };
  
  return { valid: true, sanitizedName };
}
```

#### Query Validation

```typescript
function validateQuery(query: string): ValidationResult {
  // Length check
  if (query.length === 0 || query.length > 500) {
    return { valid: false, error: 'INVALID_QUERY_LENGTH' };
  }
  
  // SQL injection prevention (parameterized queries)
  // XSS prevention (sanitize before display)
  const sanitized = sanitizeHtml(query, {
    allowedTags: [],
    allowedAttributes: {}
  });
  
  return { valid: true, sanitized };
}
```

#### Structured Data Validation

```typescript
function validateStructuredData(data: any): ValidationResult {
  const schema = {
    amount: { type: 'number', min: 0, max: 10000000 },
    vendor: { type: 'string', maxLength: 200 },
    chargeType: { type: 'enum', values: ['mrp', 'service_charge', 'challan', 'other'] },
    date: { type: 'date', min: '2000-01-01', max: 'today+1' }
  };
  
  // Validate against schema
  // Return validation errors
}
```

### Rate Limiting

**API Gateway Throttling**:
```typescript
const rateLimits = {
  '/api/upload': {
    rateLimit: 10, // requests per minute
    burstLimit: 20
  },
  '/api/query': {
    rateLimit: 30,
    burstLimit: 50
  },
  '/api/analyze': {
    rateLimit: 20,
    burstLimit: 40
  },
  '/api/complaint': {
    rateLimit: 10,
    burstLimit: 15
  }
};
```

**IP-based Rate Limiting**:
- Track requests per IP in Redis
- Block IPs exceeding limits for 1 hour
- Whitelist for known good actors

### Security Headers

```typescript
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Content-Security-Policy': "default-src 'self'; img-src 'self' data: https:; script-src 'self' 'unsafe-inline' 'unsafe-eval'",
  'Referrer-Policy': 'strict-origin-when-cross-origin'
};
```

### Vulnerability Management

**Dependency Scanning**:
- Automated npm audit in CI/CD
- Dependabot for security updates
- Regular dependency updates

**Code Scanning**:
- Static analysis with ESLint security plugins
- SAST tools (Snyk, SonarQube)
- Regular security audits

**Penetration Testing**:
- Annual third-party penetration testing
- Bug bounty program (future)
- Responsible disclosure policy

### Compliance

**Indian Data Protection**:
- Comply with Digital Personal Data Protection Act, 2023
- Data localization (store data in India region)
- User consent management
- Right to deletion

**Consumer Protection**:
- Accurate legal information
- Clear disclaimers
- No misleading claims
- Transparent data usage

## Deployment Architecture

### Infrastructure as Code

**AWS CDK Stack**:
```typescript
export class HaqDarshakStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);
    
    // S3 Bucket for documents
    const documentBucket = new s3.Bucket(this, 'DocumentBucket', {
      encryption: s3.BucketEncryption.S3_MANAGED,
      lifecycleRules: [
        {
          expiration: Duration.days(30),
          id: 'DeleteOldDocuments'
        }
      ],
      cors: [
        {
          allowedMethods: [s3.HttpMethods.POST, s3.HttpMethods.GET],
          allowedOrigins: ['https://haqdarshak.ai'],
          allowedHeaders: ['*']
        }
      ]
    });
    
    // Lambda Functions
    const ocrLambda = new lambda.Function(this, 'OCRLambda', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'ocr.handler',
      code: lambda.Code.fromAsset('lambda/ocr'),
      timeout: Duration.seconds(30),
      memorySize: 2048,
      environment: {
        BUCKET_NAME: documentBucket.bucketName
      }
    });
    
    // API Gateway
    const api = new apigateway.RestApi(this, 'HaqDarshakAPI', {
      restApiName: 'HaqDarshak API',
      deployOptions: {
        throttlingRateLimit: 100,
        throttlingBurstLimit: 200
      }
    });
    
    // ... additional resources
  }
}
```

### CI/CD Pipeline

**GitHub Actions Workflow**:
```yaml
name: Deploy HaqDarshak AI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run test:property
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - run: npm ci
      - run: npm run build
      - run: npx cdk deploy --require-approval never
```

### Monitoring and Observability

**CloudWatch Metrics**:
- Lambda invocation count, duration, errors
- API Gateway request count, latency, 4xx/5xx errors
- S3 bucket size, request count
- Bedrock API usage and costs

**CloudWatch Alarms**:
- Lambda error rate > 5%
- API latency > 10 seconds
- S3 upload failures
- Bedrock throttling

**Logging**:
- Structured JSON logs
- Correlation IDs for request tracing
- Log aggregation in CloudWatch Logs
- Log retention: 30 days

**Distributed Tracing**:
- AWS X-Ray for request tracing
- Trace Lambda cold starts
- Identify performance bottlenecks

### Disaster Recovery

**Backup Strategy**:
- Database: Daily automated backups (Supabase)
- S3: Versioning enabled
- Legal rules: Version controlled in Git

**Recovery Objectives**:
- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 24 hours

**Failover Plan**:
1. Switch to backup Bedrock region if primary fails
2. Serve cached legal rules if database unavailable
3. Queue uploads if S3 unavailable
4. Display maintenance page if all services down

## Performance Optimization

### Caching Strategy

**Multi-Level Caching**:

1. **Browser Cache** (Service Worker):
   - Cache static assets (JS, CSS, images)
   - Cache legal rules for offline access
   - Cache user's language preference

2. **CDN Cache** (CloudFront):
   - Cache static frontend assets
   - Cache API responses for common queries
   - Edge locations in India for low latency

3. **Application Cache** (Redis):
   - Cache legal rules (24 hour TTL)
   - Cache OCR results (1 hour TTL)
   - Cache Bedrock embeddings (query hash → embedding)

4. **Database Query Cache**:
   - Supabase query caching
   - Materialized views for analytics

### Lambda Optimization

**Cold Start Reduction**:
- Provisioned concurrency for critical Lambdas
- Minimize deployment package size
- Use Lambda layers for shared dependencies
- Keep functions warm with scheduled pings

**Memory Allocation**:
- OCR Lambda: 2048 MB (CPU-intensive)
- Data Extraction Lambda: 1024 MB
- RAG Query Lambda: 1024 MB
- Complaint Generation Lambda: 512 MB

**Parallel Processing**:
```typescript
async function analyzeDocument(documentId: string) {
  // Run OCR and legal rule loading in parallel
  const [ocrResult, legalRules] = await Promise.all([
    invokeOCRLambda(documentId),
    loadLegalRules() // From cache
  ]);
  
  // Then run extraction and RAG in sequence
  const structuredData = await extractData(ocrResult.text);
  const matchedRules = await queryRAG(structuredData, legalRules);
  
  return { structuredData, matchedRules };
}
```

### Database Optimization

**Indexing Strategy**:
```sql
-- Frequently queried fields
CREATE INDEX idx_documents_session ON documents(user_session_id);
CREATE INDEX idx_documents_uploaded ON documents(uploaded_at);
CREATE INDEX idx_analyses_document ON analyses(document_id);

-- Full-text search on legal rules
CREATE INDEX idx_legal_rules_description ON legal_rules USING GIN(to_tsvector('english', description));

-- Composite index for common queries
CREATE INDEX idx_analyses_session_date ON analyses(user_session_id, created_at DESC);
```

**Connection Pooling**:
```typescript
const supabase = createClient(SUPABASE_URL, SUPABASE_KEY, {
  db: {
    poolSize: 10,
    idleTimeoutMillis: 30000
  }
});
```

### Frontend Optimization

**Code Splitting**:
```typescript
// Lazy load heavy components
const ComplaintGenerator = lazy(() => import('./ComplaintGenerator'));
const PDFViewer = lazy(() => import('./PDFViewer'));
```

**Image Optimization**:
- Next.js Image component for automatic optimization
- WebP format with JPEG fallback
- Responsive images for different screen sizes
- Lazy loading for below-the-fold images

**Bundle Size Reduction**:
- Tree shaking unused code
- Dynamic imports for routes
- Minimize third-party dependencies
- Use lightweight alternatives (e.g., date-fns instead of moment.js)

## Internationalization (i18n)

### Translation Management

**Translation Files**:
```typescript
// locales/en.json
{
  "landing": {
    "title": "Know Your Rights, Protect Your Money",
    "uploadButton": "Upload Bill",
    "queryButton": "Ask Question"
  },
  "verdict": {
    "violation": "Illegal Charge Detected",
    "legal": "Charge Appears Legal",
    "disclaimer": "This is for informational purposes only..."
  }
}

// locales/hi.json
{
  "landing": {
    "title": "अपने अधिकार जानें, अपने पैसे बचाएं",
    "uploadButton": "बिल अपलोड करें",
    "queryButton": "प्रश्न पूछें"
  },
  "verdict": {
    "violation": "अवैध शुल्क का पता चला",
    "legal": "शुल्क कानूनी प्रतीत होता है",
    "disclaimer": "यह केवल सूचनात्मक उद्देश्यों के लिए है..."
  }
}
```

**Translation Hook**:
```typescript
function useTranslation() {
  const { language } = useLanguage();
  const translations = language === 'hi' ? hiTranslations : enTranslations;
  
  const t = (key: string) => {
    const keys = key.split('.');
    let value = translations;
    for (const k of keys) {
      value = value[k];
    }
    return value || key;
  };
  
  return { t, language };
}
```

### Number and Date Formatting

```typescript
function formatCurrency(amount: number, language: string): string {
  return new Intl.NumberFormat(language === 'hi' ? 'hi-IN' : 'en-IN', {
    style: 'currency',
    currency: 'INR'
  }).format(amount);
}

function formatDate(date: Date, language: string): string {
  return new Intl.DateTimeFormat(language === 'hi' ? 'hi-IN' : 'en-IN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  }).format(date);
}
```

## Future Enhancements

### Phase 2 Features

1. **Expanded Legal Coverage**:
   - GST violations
   - Banking charges
   - Insurance mis-selling
   - Real estate violations

2. **User Accounts**:
   - Save complaint history
   - Track complaint status
   - Receive notifications

3. **Government Integration**:
   - Direct complaint filing to authorities
   - Real-time status updates
   - Digital signatures

4. **Community Features**:
   - Share violations anonymously
   - Crowdsourced violation database
   - Merchant ratings

5. **Advanced Analytics**:
   - Violation trends by region
   - Merchant violation history
   - Success rate of complaints

### Scalability Roadmap

**10K Users**:
- Current architecture sufficient
- Basic monitoring

**100K Users**:
- Add Redis caching
- Increase Lambda concurrency limits
- CDN for static assets

**1M Users**:
- Multi-region deployment
- Read replicas for database
- Advanced caching strategies

**10M+ Users**:
- Microservices architecture
- Event-driven processing
- Dedicated OCR infrastructure
- Multi-tenant database sharding

