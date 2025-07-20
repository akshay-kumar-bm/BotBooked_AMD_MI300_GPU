# BotBooked AI Scheduling Assistant

An intelligent AI-powered scheduling assistant that autonomously schedules meetings by analyzing calendar availability, user preferences, and meeting priorities. Built for the AMD AI Hackathon using AMD MI300 GPU acceleration.

## 🎯 Overview

BotBooked is an agentic AI system that eliminates the back-and-forth of meeting coordination by automatically:
- **Understanding natural language** meeting requests using advanced LLM processing
- **Analyzing calendar availability** across multiple attendees via Google Calendar integration
- **Resolving scheduling conflicts** intelligently using priority-based rescheduling
- **Respecting user preferences** for working hours, meeting limits, and buffer times
- **Providing fallback strategies** when initial scheduling attempts fail

### Key Features

✅ **Autonomous Coordination** - Schedules meetings without human micromanagement  
✅ **Dynamic Adaptability** - Handles last-minute changes and conflicting priorities  
✅ **Natural Language Processing** - Understands requests like "Let's meet Thursday for 30 minutes"  
✅ **Priority-Based Scheduling** - Higher priority meetings can displace lower priority ones  
✅ **User Preference Awareness** - Respects working hours, daily limits, and buffer requirements  
✅ **Intelligent Rescheduling** - Automatically finds new slots for displaced meetings  
✅ **Multiple Fallback Strategies** - Shorter durations, reduced attendees, shifted times  
✅ **Fast Response Times** - Optimized for <10 second latency requirements  

## 🏗️ Architecture

The system uses a multi-stage workflow orchestrated by LangGraph:

```
Email Request → LLM Parsing → Calendar Analysis → Scheduling Algorithm → Conflict Resolution → JSON Response
```

### Core Components

1. **LLM Integration** - Uses LangChain with vLLM server for natural language understanding
2. **Google Calendar API** - OAuth-based calendar access and event management  
3. **Scheduling Engine** - Two-phase algorithm (free slots → reschedulable slots)
4. **Preference System** - User-specific working hours and meeting constraints
5. **Priority Manager** - 4-level priority system (P1=highest to P4=lowest)
6. **Conflict Resolver** - Recursive rescheduling with cascading conflict handling

## 🚀 Installation

### Prerequisites

- Python 3.8+
- AMD MI300 GPU with ROCm support
- Google Calendar API credentials
- vLLM server setup

### Dependencies

```bash
pip install langchain langchain-openai langgraph
pip install google-auth google-auth-oauthlib google-auth-httplib2
pip install google-api-python-client
pip install flask typing-extensions
```

### GPU Setup (AMD MI300)

1. **Install ROCm** following [AMD ROCm documentation](https://rocmdocs.amd.com/)

2. **Setup vLLM Server with DeepSeek Model**:
```bash
HIP_VISIBLE_DEVICES=0 vllm serve /home/user/Models/deepseek-ai/deepseek-llm-7b-chat \
    --gpu-memory-utilization 0.9 \
    --swap-space 16 \
    --disable-log-requests \
    --dtype float16 \
    --max-model-len 2048 \
    --tensor-parallel-size 1 \
    --host 0.0.0.0 \
    --port 3000 \
    --num-scheduler-steps 10 \
    --max-num-seqs 128 \
    --max-num-batched-tokens 2048 \
    --distributed-executor-backend "mp"
```

3. **Alternative: Llama Model Setup**:
```bash
HIP_VISIBLE_DEVICES=0 vllm serve /home/user/Models/meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.3 \
    --port 4000 \
    [... other parameters same as above]
```

### Google Calendar Setup

1. **Create Google Cloud Project** and enable Calendar API
2. **Setup OAuth credentials** and download `credentials.json`
3. **Place OAuth tokens** in `Keys/` directory:
   ```
   Keys/
   ├── userone.token
   ├── usertwo.token  
   └── userthree.token
   ```

## 📖 Usage

### Basic API Usage

1. **Start the Flask Server**:
```python
from app import builder
from flask import Flask, request, jsonify

app = Flask(__name__)
graph = builder.compile()

@app.route('/receive', methods=['POST'])
def receive():
    data = request.get_json()
    result = graph.invoke({"user_request": data})
    return jsonify(result['final_output'])

app.run(host='0.0.0.0', port=5000)
```

2. **Send Meeting Request**:
```python
import requests
import json

# Example meeting request
request_data = {
    "Request_id": "unique-id-123",
    "Datetime": "19-07-2025T12:34:55",
    "Location": "Conference Room A",
    "From": "userone.amd@gmail.com",
    "Attendees": [
        {"email": "usertwo.amd@gmail.com"},
        {"email": "userthree.amd@gmail.com"}
    ],
    "Subject": "Project Status Update",
    "EmailContent": "Hi team, let's meet on Thursday for 30 minutes to discuss the project status."
}

# Send to scheduler
response = requests.post("http://localhost:5000/receive", json=request_data)
result = response.json()
print(json.dumps(result, indent=2))
```

### Expected Response Format

```json
{
  "Request_id": "unique-id-123",
  "Datetime": "19-07-2025T12:34:55",
  "Location": "Conference Room A", 
  "From": "userone.amd@gmail.com",
  "Attendees": [
    {
      "email": "userone.amd@gmail.com",
      "events": [
        {
          "StartTime": "2025-07-24T10:30:00+05:30",
          "EndTime": "2025-07-24T11:00:00+05:30", 
          "NumAttendees": 3,
          "Attendees": ["userone.amd@gmail.com", "usertwo.amd@gmail.com", "userthree.amd@gmail.com"],
          "Summary": "Project Status Update"
        }
      ]
    }
  ],
  "Subject": "Project Status Update",
  "EmailContent": "Hi team, let's meet on Thursday for 30 minutes...",
  "EventStart": "2025-07-24T10:30:00+05:30",
  "EventEnd": "2025-07-24T11:00:00+05:30", 
  "Duration_mins": "30",
  "MetaData": {}
}
```

## 🎛️ Configuration

### User Preferences

Modify `PARTICIPANT_PREFERENCES` in `app.py`:

```python
PARTICIPANT_PREFERENCES = {
    "user1": {
        "preferred_hours": {"start": 9, "end": 17},  # 9 AM to 5 PM
        "max_meetings_per_day": 6,
        "avoid_back_to_back": True,
        "buffer_minutes": 15
    }
}
```

### LLM Configuration

Update the model configuration in `extract_meeting_details_with_llm()`:

```python
model = ChatOpenAI(
    model="Qwen/Qwen3-4B",           # Model name
    temperature=0,                   # Deterministic output
    base_url="http://localhost:8000/v1/",  # vLLM server URL
    api_key="REDACTED"               # Placeholder for local server
)
```

### Priority System

The system uses a 4-level priority system:
- **P1 (Highest)**: Critical meetings, board meetings, client calls
- **P2 (High)**: Important project meetings, stakeholder reviews  
- **P3 (Medium)**: Regular team meetings, design reviews
- **P4 (Lowest)**: Informal sync-ups, optional meetings

## 🧠 How It Works

### 1. Natural Language Processing
- Parses email content using LLM with detailed prompts
- Resolves relative time expressions ("tomorrow", "next week")
- Extracts meeting duration, priority, and participants
- Handles ambiguous time references intelligently

### 2. Two-Phase Scheduling Algorithm

**Phase 1: Free Slot Search**
- Merges all attendees' busy time slots
- Finds gaps between existing meetings
- Validates against working hours and preferences
- Calculates preference violation scores

**Phase 2: Priority-Based Rescheduling**
- If no free slots, examines existing meetings
- Only displaces lower-priority meetings
- Ensures all conflicts are resolvable
- Triggers recursive rescheduling for displaced meetings

### 3. Preference Scoring System

The system calculates penalty scores for constraint violations:
- **Working Hours Violation**: +50 points per person
- **Daily Meeting Limit Exceeded**: +30 points per person  
- **Back-to-Back Meeting Conflict**: +20 points per person

### 4. Fallback Strategies

When initial scheduling fails, the system tries:
1. **Shorter Duration** (75% of original)
2. **Majority Attendees** (if >2 attendees)
3. **Time Shifting** (±30, ±60 minutes)
4. **Relaxed Preferences** (higher violation tolerance)

## 📂 Project Structure

```
BotBooked_AMD_MI300_GPU/
├── app.py                 # Main scheduling engine
├── Submission.ipynb      # Flask server setup
├── README.md             # This file
├── JSON_Samples/         # Example input/output
│   ├── Input_Request.json
│   └── Output_Event.json
├── Keys/                 # OAuth tokens (create this)
├── notebook/             # Development notebooks
└── AI-Scheduling-Assistant/  # Setup documentation
```

## 🔧 Key Functions

### Core Scheduling
- `find_earliest_slot()` - Two-phase scheduling algorithm
- `schedule_meeting_from_request()` - High-level scheduling coordinator
- `reschedule_meetings_recursively()` - Intelligent conflict resolution

### Calendar Integration  
- `retrive_calendar_events()` - Google Calendar API interface
- `parse_calendar_data()` - Calendar data validation and processing

### Natural Language Processing
- `extract_meeting_details_with_llm()` - LLM-based email parsing
- Comprehensive prompt engineering with examples

### Preference Management
- `get_user_preferences()` - User preference lookup
- `calculate_preference_score()` - Constraint violation scoring

## ⚡ Performance

- **Target Latency**: <10 seconds end-to-end
- **GPU Acceleration**: AMD MI300 with ROCm optimizations
- **Concurrent Processing**: vLLM batching for multiple requests
- **Memory Optimization**: Efficient calendar data structures

## 🤝 Contributing

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. **Open** a Pull Request

### Development Guidelines
- Follow existing code documentation standards
- Add comprehensive docstrings to new functions
- Include error handling and input validation
- Test with various meeting scenarios

## 📄 License

This project was developed for the AMD AI Hackathon. Please refer to the hackathon terms and conditions for usage rights.

## 🙏 Acknowledgments

- **AMD** for providing MI300 GPU access and support
- **LangChain/LangGraph** for the AI workflow framework
- **vLLM** for high-performance LLM inference
- **Google Calendar API** for calendar integration

## 📞 Support

For issues and questions:
1. Check the [notebook examples](./notebook/) for detailed usage
2. Review the comprehensive function documentation in `app.py`
3. Examine [JSON_Samples/](./JSON_Samples/) for request/response formats

---

**Built with ❤️ for the AMD AI Hackathon using MI300 GPU acceleration**