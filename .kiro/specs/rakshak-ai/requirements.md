# Requirements Document: HaqDarshak AI

## Introduction

HaqDarshak AI is an AI-powered legal violation detector designed for Indian citizens to instantly verify if they're being illegally charged by scanning bills, challans, or receipts. The system addresses the core problem of citizens losing money daily due to illegal charges (MRP overcharging, unauthorized service charges, inflated traffic fines) because they lack knowledge of their legal rights at the moment of transaction.

The solution provides instant verification against Indian consumer protection laws, generates ready-to-send legal complaints, and empowers citizens with actionable enforcement tools.

## Glossary

- **System**: The HaqDarshak AI application
- **User**: Indian citizen using the application to verify charges
- **OCR_Engine**: Optical Character Recognition component for text extraction
- **RAG_System**: Retrieval-Augmented Generation system for legal rule matching
- **Legal_Database**: Curated collection of Indian consumer protection laws and rules
- **Bedrock_Service**: Amazon Bedrock AI service using Claude Sonnet 4.5
- **Violation_Detector**: Component that analyzes charges against legal rules
- **Complaint_Generator**: Component that creates structured legal complaints
- **Document**: Uploaded image file (receipt, bill, challan)
- **Verdict**: System output indicating whether a charge is legal or illegal
- **Authority**: Government body responsible for enforcement (CCPA, Legal Metrology, RTO)
- **MRP**: Maximum Retail Price as defined by Legal Metrology Act
- **CCPA**: Central Consumer Protection Authority
- **Service_Charge**: Restaurant charge for service (subject to CCPA guidelines)
- **Challan**: Traffic violation fine notice
- **Structured_Data**: Extracted information (amount, vendor, charge type, date)

## Requirements

### Requirement 1: Bilingual Text Query Input

**User Story:** As a user, I want to ask questions about charges in Hindi or English, so that I can verify legality in my preferred language.

#### Acceptance Criteria

1. WHEN a user submits a text query in English, THE System SHALL process the query and return results in English
2. WHEN a user submits a text query in Hindi, THE System SHALL process the query and return results in Hindi
3. WHEN a user submits a mixed-language query, THE System SHALL detect the primary language and respond accordingly
4. THE System SHALL support text queries up to 500 characters in length
5. WHEN a query contains insufficient information, THE System SHALL prompt the user for missing details

### Requirement 2: Document Upload and Processing

**User Story:** As a user, I want to upload images of bills, receipts, or challans, so that the system can automatically extract and verify charges.

#### Acceptance Criteria

1. WHEN a user uploads a JPG file, THE System SHALL accept and process the file
2. WHEN a user uploads a PNG file, THE System SHALL accept and process the file
3. WHEN a user uploads a PDF file, THE System SHALL accept and process the file
4. WHEN a user uploads a file larger than 10MB, THE System SHALL reject the upload and display an error message
5. WHEN a user uploads an unsupported file format, THE System SHALL reject the upload and display an error message
6. THE System SHALL store uploaded documents securely in AWS S3 with encryption
7. WHEN a document is uploaded, THE System SHALL complete processing within 10 seconds

### Requirement 3: OCR Text Extraction

**User Story:** As a user, I want the system to automatically extract text from my uploaded documents, so that I don't have to manually type information.

#### Acceptance Criteria

1. WHEN a document is uploaded, THE OCR_Engine SHALL extract all visible text from the image
2. THE OCR_Engine SHALL achieve at least 85% accuracy on printed text
3. WHEN OCR extraction fails or produces low-confidence results, THE System SHALL provide a manual entry fallback option
4. THE OCR_Engine SHALL extract text from documents in Hindi and English
5. WHEN text extraction is complete, THE System SHALL return the raw extracted text

### Requirement 4: Structured Data Extraction

**User Story:** As a user, I want the system to identify key information from my documents, so that charges can be verified against legal rules.

#### Acceptance Criteria

1. WHEN raw text is available, THE System SHALL extract the total amount charged
2. WHEN raw text is available, THE System SHALL extract the vendor or merchant name
3. WHEN raw text is available, THE System SHALL extract the charge type (MRP, service charge, fine amount)
4. WHEN raw text is available, THE System SHALL extract the transaction date
5. WHEN raw text is available, THE System SHALL extract product names and individual prices for MRP verification
6. WHEN extraction confidence is below 70%, THE System SHALL flag fields for user verification
7. THE System SHALL structure extracted data in JSON format with labeled fields

### Requirement 5: Legal Rule Matching via RAG

**User Story:** As a user, I want the system to check my charges against actual Indian laws, so that I receive accurate legal information.

#### Acceptance Criteria

1. WHEN structured data is available, THE RAG_System SHALL query the Legal_Database for relevant rules
2. THE RAG_System SHALL match charges against Legal Metrology Act rules for MRP violations
3. THE RAG_System SHALL match charges against CCPA guidelines for service charge violations
4. THE RAG_System SHALL match charges against Motor Vehicles Act rules for challan violations
5. THE RAG_System SHALL return only rules from the curated Legal_Database without generating new legal interpretations
6. WHEN multiple rules apply, THE RAG_System SHALL return all applicable rules ranked by relevance
7. THE RAG_System SHALL use Amazon Bedrock with Claude Sonnet 4.5 for semantic matching

### Requirement 6: Violation Detection and Verdict Generation

**User Story:** As a user, I want to receive a clear verdict on whether charges are legal or illegal, so that I can take appropriate action.

#### Acceptance Criteria

1. WHEN legal rules are matched, THE Violation_Detector SHALL determine if a violation exists
2. WHEN a violation is detected, THE System SHALL generate a verdict stating "Illegal Charge Detected"
3. WHEN no violation is detected, THE System SHALL generate a verdict stating "Charge Appears Legal"
4. WHEN insufficient information exists, THE System SHALL generate a verdict stating "Unable to Determine - More Information Needed"
5. THE System SHALL include specific law citations (Act name, section number) in every verdict
6. THE System SHALL include a brief explanation of why the charge is legal or illegal
7. THE System SHALL display a legal disclaimer stating "This is for informational purposes only and does not constitute legal advice"

### Requirement 7: MRP Overcharging Detection

**User Story:** As a shopper, I want to verify if a store charged me above MRP, so that I can report violations to authorities.

#### Acceptance Criteria

1. WHEN a product price and MRP are extracted, THE System SHALL compare the charged price against the MRP
2. WHEN charged price exceeds MRP, THE System SHALL flag this as a violation of Legal Metrology Act Section 18(1)
3. WHEN MRP is not visible in the document, THE System SHALL request manual MRP entry
4. THE System SHALL account for taxes included in MRP as per legal requirements
5. THE System SHALL identify the appropriate authority (Legal Metrology Department) for MRP violations

### Requirement 8: Restaurant Service Charge Validation

**User Story:** As a restaurant customer, I want to verify if service charges are mandatory, so that I can refuse payment if they're voluntary.

#### Acceptance Criteria

1. WHEN a service charge line item is detected, THE System SHALL check if it was presented as mandatory
2. THE System SHALL cite CCPA guidelines stating service charges must be voluntary
3. WHEN a service charge is marked as mandatory or automatically added, THE System SHALL flag this as a violation
4. THE System SHALL provide information on the customer's right to refuse service charges
5. THE System SHALL identify CCPA as the appropriate authority for service charge complaints

### Requirement 9: Traffic Challan Verification

**User Story:** As a driver, I want to verify if my traffic fine amount is correct, so that I can challenge inflated challans.

#### Acceptance Criteria

1. WHEN a challan document is uploaded, THE System SHALL extract the violation type and fine amount
2. THE System SHALL match the fine amount against Motor Vehicles Act penalty schedules
3. WHEN the fine amount exceeds the legal maximum, THE System SHALL flag this as an invalid challan
4. THE System SHALL provide the correct fine amount as per Motor Vehicles Act
5. THE System SHALL identify the RTO or Traffic Police as the appropriate authority for challan disputes

### Requirement 10: Instant Complaint Generation

**User Story:** As a user who discovered a violation, I want to generate a ready-to-send complaint, so that I can quickly report the issue to authorities.

#### Acceptance Criteria

1. WHEN a violation is detected, THE Complaint_Generator SHALL create a structured complaint document
2. THE Complaint_Generator SHALL include all extracted transaction details in the complaint
3. THE Complaint_Generator SHALL include relevant law citations and sections
4. THE Complaint_Generator SHALL address the complaint to the appropriate authority
5. THE Complaint_Generator SHALL include user-editable fields for contact information
6. THE System SHALL generate complaints in both Hindi and English based on user preference
7. WHEN complaint generation is complete, THE System SHALL provide the complaint within 3 seconds

### Requirement 11: Multi-Channel Complaint Delivery

**User Story:** As a user, I want to send my complaint through multiple channels, so that I can choose the most convenient method.

#### Acceptance Criteria

1. WHEN a complaint is generated, THE System SHALL provide a WhatsApp-ready message template
2. WHEN a complaint is generated, THE System SHALL provide an email draft with pre-filled subject and body
3. WHEN a complaint is generated, THE System SHALL provide a downloadable PDF version
4. THE System SHALL format WhatsApp messages to fit within character limits while preserving key information
5. THE System SHALL include authority contact information in email drafts where available
6. THE System SHALL generate PDF complaints with proper formatting and legal structure

### Requirement 12: Mobile-First User Interface

**User Story:** As a smartphone user, I want a responsive interface optimized for mobile devices, so that I can easily use the app on my phone.

#### Acceptance Criteria

1. THE System SHALL display correctly on screen sizes from 320px to 428px width
2. THE System SHALL provide touch-optimized buttons with minimum 44px touch targets
3. WHEN on mobile devices, THE System SHALL use the device camera for direct document capture
4. THE System SHALL load the landing page within 3 seconds on 4G connections
5. THE System SHALL use progressive image loading for uploaded documents
6. THE System SHALL provide clear visual feedback during OCR and analysis processing

### Requirement 13: Performance and Response Time

**User Story:** As a user, I want fast results, so that I can make decisions at the point of transaction.

#### Acceptance Criteria

1. WHEN a document is uploaded, THE System SHALL complete OCR extraction within 5 seconds
2. WHEN structured data is extracted, THE System SHALL complete legal analysis within 5 seconds
3. THE System SHALL return a complete verdict with complaint options within 10 seconds of upload
4. WHEN the Bedrock_Service experiences latency, THE System SHALL display a progress indicator
5. THE System SHALL cache frequently accessed legal rules to reduce query time

### Requirement 14: Data Security and Privacy

**User Story:** As a user, I want my personal and financial information protected, so that my privacy is maintained.

#### Acceptance Criteria

1. THE System SHALL encrypt all uploaded documents at rest in AWS S3
2. THE System SHALL encrypt all data in transit using TLS 1.3
3. THE System SHALL not store user contact information without explicit consent
4. WHEN a user deletes a document, THE System SHALL permanently remove it from storage within 24 hours
5. THE System SHALL anonymize extracted data used for system improvement
6. THE System SHALL comply with Indian data protection regulations
7. THE System SHALL provide a privacy policy accessible from all pages

### Requirement 15: Legal Database Management

**User Story:** As a system administrator, I want to maintain accurate legal rules, so that users receive correct information.

#### Acceptance Criteria

1. THE Legal_Database SHALL store rules in JSON format with standardized schema
2. WHEN a legal rule is added, THE System SHALL validate the rule structure against the schema
3. THE Legal_Database SHALL include fields for rule_id, category, law, section, description, violation_keywords, penalty, and authority
4. THE Legal_Database SHALL support versioning of legal rules for audit trails
5. WHEN a rule is updated, THE System SHALL timestamp the change and preserve the previous version

### Requirement 16: Error Handling and Fallbacks

**User Story:** As a user, I want clear error messages and alternatives when something goes wrong, so that I can still accomplish my goal.

#### Acceptance Criteria

1. WHEN OCR fails, THE System SHALL provide a manual text entry form
2. WHEN the Bedrock_Service is unavailable, THE System SHALL display a retry option and estimated wait time
3. WHEN network connectivity is lost, THE System SHALL save the user's progress locally
4. WHEN an uploaded document is unreadable, THE System SHALL suggest image quality improvements
5. WHEN legal rules cannot be matched, THE System SHALL provide general guidance and suggest contacting a legal expert
6. THE System SHALL log all errors with sufficient context for debugging

### Requirement 17: Bilingual Content Display

**User Story:** As a Hindi-speaking user, I want all interface elements and results in Hindi, so that I can fully understand the information.

#### Acceptance Criteria

1. THE System SHALL provide a language toggle between Hindi and English on all pages
2. WHEN Hindi is selected, THE System SHALL display all UI labels, buttons, and messages in Hindi
3. WHEN Hindi is selected, THE System SHALL generate verdicts and complaints in Hindi
4. THE System SHALL translate legal citations to Hindi while preserving English Act names in parentheses
5. THE System SHALL persist language preference across user sessions

### Requirement 18: Landing Page and User Onboarding

**User Story:** As a new user, I want to quickly understand what the app does and how to use it, so that I can start verifying charges immediately.

#### Acceptance Criteria

1. THE System SHALL display a landing page with "Upload Bill" and "Ask Question" call-to-action buttons
2. THE System SHALL provide a brief explanation of the three primary use cases on the landing page
3. THE System SHALL display example scenarios with before/after screenshots
4. WHEN a user first visits, THE System SHALL show a quick tutorial overlay explaining the process
5. THE System SHALL provide a "Skip Tutorial" option for returning users

### Requirement 19: Results Display and Action Options

**User Story:** As a user who received a verdict, I want to see all relevant information and available actions, so that I can decide my next steps.

#### Acceptance Criteria

1. WHEN analysis is complete, THE System SHALL display the verdict prominently at the top of the results page
2. THE System SHALL display all extracted structured data with edit options
3. THE System SHALL display applicable law citations with expandable explanations
4. WHEN a violation is detected, THE System SHALL display action buttons for complaint generation
5. WHEN no violation is detected, THE System SHALL display educational information about the relevant law
6. THE System SHALL provide a "Share Results" option for social media or messaging apps

### Requirement 20: Serverless Architecture and Scalability

**User Story:** As a system architect, I want a scalable serverless architecture, so that the system can handle millions of users without infrastructure management.

#### Acceptance Criteria

1. THE System SHALL use AWS Lambda for all backend processing functions
2. THE System SHALL use API Gateway for RESTful API endpoints
3. THE System SHALL auto-scale Lambda functions based on request volume
4. THE System SHALL use Supabase PostgreSQL for structured data storage with connection pooling
5. THE System SHALL implement request throttling to prevent abuse
6. WHEN traffic spikes occur, THE System SHALL maintain response times within acceptable limits
