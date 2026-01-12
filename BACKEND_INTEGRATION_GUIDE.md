# 🔧 Backend Integration Guide for Victorian Quiz

## 📋 Overview

The Victorian Personality Quiz is now designed to work with a **backend API** for result storage and retrieval. The QR code links to actual backend-hosted result pages, not client-side PDFs.

---

## 🏗️ Architecture

```
User Completes Quiz
        ↓
Frontend sends result data via API
        ↓
Backend saves to database
        ↓
Backend returns unique result ID & URL
        ↓
Frontend generates QR code with that URL
        ↓
User scans QR code
        ↓
Opens backend-hosted result page
```

---

## 📡 API Endpoints Required

### 1. **POST /api/results**
**Purpose:** Save quiz result and return unique URL

**Request Body:**
```json
{
  "character": "Jane Eyre",
  "subtitle": "The Moral Compass",
  "description": "Your character mirrors...",
  "traits": ["Principled", "Resilient", "Independent", "Passionate"],
  "scanMood": "Contemplative",
  "answers": [
    {
      "question": "When faced with injustice...",
      "answer": "I speak out firmly...",
      "traits": ["principled", "courageous"]
    }
  ],
  "physiognomySnapshot": "data:image/jpeg;base64,/9j/4AAQ...",
  "timestamp": "2026-01-08T19:42:20.000Z"
}
```

**Response:** 
```json
{
  "resultId": "vct_abc123xyz789",
  "resultUrl": "/results/vct_abc123xyz789"
}
```

**Status Codes:**
- `201 Created` - Success
- `400 Bad Request` - Invalid data
- `500 Internal Server Error` - Server error

### 2. **GET /results/{resultId}**
**Purpose:** Display stored result page

**Response:** HTML page showing:
- Character name, subtitle, description
- Distinguishing traits
- Physiognomy snapshot
- Quiz answers
- Victorian styling

---

## 💾 Database Schema

### Table: `personality_results`

```sql
CREATE TABLE personality_results (
  id VARCHAR(255) PRIMARY KEY,
  character VARCHAR(255) NOT NULL,
  subtitle VARCHAR(255),
  description TEXT,
  traits JSON,  -- Array of trait strings
  scan_mood VARCHAR(100),
  answers JSON,  -- Array of Q&A objects
  physiognomy_snapshot TEXT,  -- Base64 image data
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_created_at (created_at)
);
```

**Example Row:**
```json
{
  "id": "vct_abc123xyz789",
  "character": "Jane Eyre",
  "subtitle": "The Moral Compass",
  "description": "Your character mirrors...",
  "traits": ["Principled", "Resilient", "Independent", "Passionate"],
  "scan_mood": "Contemplative",
  "answers": [...],
  "physiognomy_snapshot": "data:image/jpeg;base64,...",
  "created_at": "2026-01-08 19:42:20"
}
```

---

## 🔐 Security Considerations

### ID Generation
- Use cryptographically secure random IDs
- Format: `vct_` + 16 random alphanumeric characters
- Example: `vct_k7n2m9p4x8q1w5z3`

```javascript
// Node.js example
const crypto = require('crypto');
const generateResultId = () => {
  return 'vct_' + crypto.randomBytes(12).toString('hex');
};
```

###  Rate Limiting
- Limit API calls per IP: 10 requests/minute
- Prevent spam and abuse

### Data Validation
- Validate all incoming data
- Sanitize HTML/SQL injection attempts
- Limit image size (max 2MB)

### Privacy
- No personally identifiable information collected
- Results accessible only via unique URL
- Optional: Add expiration (delete after 90 days)

---

## 🎨 Backend Result Page Design

Your `/results/{id}` page should match the Victorian aesthetic:

### HTML Structure
```html
<!DOCTYPE html>
<html>
<head>
  <title>Your Victorian Analysis - {Character}</title>
  <!-- Victorian CSS styles -->
</head>
<body>
  <div class="result-hero">
    <h1>{Character Name}</h1>
    <p class="subtitle">"{Subtitle}"</p>
  </div>
  
  <div class="physiognomy-section">
    <h2>Physiognomy Analysis</h2>
    <img src="{snapshot_base64}" alt="Your portrait">
    <p>Disposition: {scan_mood}</p>
  </div>
  
  <div class="character-analysis">
    <h2>Character Analysis</h2>
    <p>{description}</p>
  </div>
  
  <div class="traits">
    <h2>Distinguishing Traits</h2>
    <ul>
      {#each trait}
        <li>{trait}</li>
      {/each}
    </ul>
  </div>
  
  <div class="responses">
    <h2>Your Responses</h2>
    {#each answer}
      <div class="qa-pair">
        <strong>Q:</strong> {question}
        <p><strong>A:</strong> {answer}</p>
      </div>
    {/each}
  </div>
  
  <footer>
    <p>Result ID: {id}</p>
    <p>Generated: {timestamp}</p>
    <button onclick="downloadPDF()">Download PDF</button>
    <button onclick="shareResult()">Share</button>
  </footer>
</body>
</html>
```

### CSS Styling
Use the same Victorian color scheme:
- Gold: `#b88643`
- Dark Brown: `#2a1f18`
- Cream: `#e8dcc8`
- Background: `#120c09`

---

## 📱 QR Code Implementation

### Frontend Code (Already Implemented)
```javascript
async function saveResultToBackend(result) {
  const apiEndpoint = '/api/results';
  
  const response = await fetch(apiEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      character: result.name,
      // ... other data
    })
  });
  
  const { resultId, resultUrl } = await response.json();
  
  // Generate QR code with backend URL
  generateQRCode(resultUrl);
}
```

### Backend Implementation Examples

#### Node.js/Express
```javascript
const express = require('express');
const app = express();

app.post('/api/results', async (req, res) => {
  const resultId = generateResultId();
  
  await db.query(
    'INSERT INTO personality_results SET ?',
    {
      id: resultId,
      character: req.body.character,
      subtitle: req.body.subtitle,
      // ... other fields
    }
  );
  
  res.status(201).json({
    resultId: resultId,
    resultUrl: `/results/${resultId}`
  });
});

app.get('/results/:id', async (req, res) => {
  const result = await db.query(
    'SELECT * FROM personality_results WHERE id = ?',
    [req.params.id]
  );
  
  if (!result) {
    return res.status(404).send('Result not found');
  }
  
  res.render('result-page', { result });
});
```

#### Python/Flask
```python
from flask import Flask, request, jsonify, render_template
import secrets

app = Flask(__name__)

@app.route('/api/results', methods=['POST'])
def save_result():
    result_id = 'vct_' + secrets.token_hex(12)
    data = request.json
    
    # Save to database
    db.execute(
        "INSERT INTO personality_results VALUES (?, ?, ?, ...)",
        result_id, data['character'], data['subtitle'], ...
    )
    
    return jsonify({
        'resultId': result_id,
        'resultUrl': f'/results/{result_id}'
    }), 201

@app.route('/results/<result_id>')
def view_result(result_id):
    result = db.execute(
        "SELECT * FROM personality_results WHERE id = ?",
        result_id
    ).fetchone()
    
    if not result:
        return "Result not found", 404
    
    return render_template('result.html', result=result)
```

---

## 🧪 Testing

### With Backend
1. Start your backend server
2. Update `apiEndpoint` in HTML to your backend URL
3. Complete quiz
4. Check console for: `✅ Result saved to backend!`
5. Scan QR code
6. Should open your `/results/{id}` page

### Without Backend (Fallback Mode)
The code includes a fallback that:
- Generates mock result IDs
- Stores in localStorage
- Still generates QR codes
- Warns in console: `⚠️ Using fallback URL`

---

## 📊 Optional: PDF Generation on Backend

### Why Backend PDF?
- Smaller QR codes
- Faster scanning
- Better security
- Consistent styling

### Implementation
```javascript
// Node.js with PDFKit
const PDFDocument = require('pdfkit');

app.get('/results/:id/pdf', async (req, res) => {
  const result = await getResult(req.params.id);
  
  const doc = new PDFDocument();
  res.setHeader('Content-Type', 'application/pdf');
  doc.pipe(res);
  
  // Add Victorian styling
  doc.fontSize(32).text('Victorian Personality Analysis');
  doc.fontSize(24).text(result.character);
  // ... add more content
  
  doc.end();
});
```

Then add download button on result page:
```html
<button onclick="window.location='/results/{id}/pdf'">
  Download PDF
</button>
```

---

##🎯 Next Steps

1. **Set up database** with the schema above
2. **Implement POST /api/results** endpoint
3. **Implement GET /results/{id}** endpoint
4. **Update frontend** `apiEndpoint` variable
5. **Style result page** with Victorian theme
6. **Test** end-to-end flow
7. **(Optional)** Add PDF generation
8. **(Optional)** Add social sharing features
9. **Deploy** to production

---

## 📝 Environment Variables

```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=victorian_quiz
DB_USER=your_username
DB_PASSWORD=your_password

# API
API_BASE_URL=https://yourdomain.com
RATE_LIMIT_PER_MINUTE=10

# Optional
RESULT_EXPIRY_DAYS=90
MAX_IMAGE_SIZE_MB=2
```

---

## ✅ Checklist

Backend Setup:
- [ ] Database created with schema
- [ ] POST /api/results implemented
- [ ] GET /results/{id} implemented
- [ ] Result page styled with Victorian theme
- [ ] Security measures in place
- [ ] Rate limiting configured

Frontend Integration:
- [ ] API endpoint configured
- [ ] Error handling in place
- [ ] Fallback mode tested
- [ ] QR code generation verified

Testing:
- [ ] Full quiz flow tested
- [ ] Result page displays correctly
- [ ] QR code scans and opens result
- [ ] Physiognomy image shows properly
- [ ] Mobile responsive

Production:
- [ ] HTTPS enabled
- [ ] CORS configured
- [ ] Database backups enabled
- [ ] Monitoring in place

---

**Status:** Frontend is ready for backend integration. Implement the backend endpoints above to enable full functionality! 🚀
