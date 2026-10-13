# Employee Attendance & Analytics API

FastAPI + MongoDB service for employee punch-in / punch-out, manual corrections with an audit trail, and analytics on attendance data.

## Requirements

- Python 3.11+
- MongoDB 6.0+ (local Docker container or a free MongoDB Atlas cluster)
- pip

## Setup

1. Clone the repository and enter the directory.

2. Create and activate a virtual environment:

   ```bash
   python -m venv venv
   # Windows
   venv\Scripts\activate
   # macOS / Linux
   source venv/bin/activate