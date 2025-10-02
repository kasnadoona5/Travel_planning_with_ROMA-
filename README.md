# Travel Planning Assistant for ROMA (Free API Optimized Setup)

This tutorial shows how to extend ROMA (installed with the kasnadoona5/ROMA_sentient optimized free-API setup) with a Travel Planning Assistant capable of generating detailed, personalized itineraries using free weather and Wikipedia data.

# (First Method)
## Overview

- Uses Open-Meteo API for free weather forecasts (no signup).
- Uses Wikipedia API for destination info.
- Runs on existing ROMA free-tier optimized LLMs: Grok-4-Fast and GLM-4.5-Air.
- Adds a new executor adapter, prompt, and configuration.
- Fully compatible with Docker deployment on your VPS.

## Step 1: Update .env with Travel APIs

Edit your `.env` file in the ROMA root folder:

```bash
nano .env
```

Add these lines:

```text
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1/forecast
WIKIPEDIA_API_URL=https://en.wikipedia.org/w/api.php
```

Save and exit.

## Step 2: Add the Travel Planner Adapter

Edit adapter file:

```bash
nano src/sentientresearchagent/hierarchical_agent_framework/agents/adapters.py
```

At the bottom, append:

```python
import os
import requests
from loguru import logger
from ..base import LlmApiAdapter

class TravelPlannerAdapter(LlmApiAdapter):
    def __init__(self, agno_agent_instance=None, agent_name="TravelPlannerAdapter"):
        super().__init__(agno_agent_instance, agent_name)
        self.weather_api = os.getenv("OPEN_METEO_BASE_URL", "https://api.open-meteo.com/v1/forecast")
        self.wiki_api = os.getenv("WIKIPEDIA_API_URL", "https://en.wikipedia.org/w/api.php")

    async def process(self, node, agent_task_input, trace_manager):
        destination = agent_task_input.current_goal.strip()
        logger.info(f"Planning trip to {destination}")

        parts = destination.split("|")
        location = parts[0].strip()
        dates = parts[1].strip() if len(parts) > 1 else "next 7 days"

        coords = self._get_coordinates(location)
        weather = {}
        if coords.get('lat') != 0:
            weather = self._get_weather(coords['lat'], coords['lon'])
        info = self._get_destination_info(location)

        return {
            "destination": location,
            "travel_dates": dates,
            "coordinates": coords,
            "weather_forecast": weather,
            "destination_info": info,
        }

    def _get_coordinates(self, location):
        try:
            params = {
                "action": "query",
                "format": "json",
                "titles": location,
                "prop": "coordinates"
            }
            resp = requests.get(self.wiki_api, params=params, timeout=10)
            data = resp.json()
            pages = data.get("query", {}).get("pages", {})
            for page in pages.values():
                coords = page.get("coordinates")
                if coords:
                    return {"lat": coords[0]["lat"], "lon": coords[0]["lon"]}
        except Exception as e:
            logger.error(f"[Wikipedia Coords] Error: {e}")
        return {"lat": 0, "lon": 0}

    def _get_weather(self, lat, lon):
        try:
            params = {
                "latitude": lat,
                "longitude": lon,
                "daily": "temperature_2m_max,temperature_2m_min,precipitation_sum,weathercode",
                "timezone": "auto"
            }
            resp = requests.get(self.weather_api, params=params, timeout=10)
            resp.raise_for_status()
            return resp.json()
        except Exception as e:
            logger.error(f"[Open-Meteo] Error: {e}")
            return {}

    def _get_destination_info(self, location):
        try:
            params = {
                "action": "query",
                "format": "json",
                "titles": location,
                "prop": "extracts",
                "exintro": True,
                "explaintext": True
            }
            resp = requests.get(self.wiki_api, params=params, timeout=10)
            data = resp.json()
            pages = data.get("query", {}).get("pages", {})
            for page in pages.values():
                return page.get("extract", "No information available")
        except Exception as e:
            logger.error(f"[Wikipedia Info] Error: {e}")
            return "Information unavailable"
```

Save and close.

## Step 3: Add Executor Prompt

Edit executor prompts file:

```bash
nano src/sentientresearchagent/hierarchical_agent_framework/agent_configs/prompts/executor_prompts.py
```

Add the following prompt (use triple quotes exactly):

```python
TRAVEL_PLANNER_EXECUTOR_SYSTEM_MESSAGE = """You are an expert travel planning assistant specialized in creating comprehensive, personalized itineraries.

### Your Task:
Analyze the provided destination data (weather, attractions, local info) and create a detailed travel itinerary.

### Provided Data Includes:
- Destination name and coordinates
- 7-day weather forecast (temperature, precipitation, conditions)
- Wikipedia destination overview
- Travel dates

### Create a Structured Response:

**Destination Overview:**
- Brief introduction to the location
- Best time to visit considerations
- Cultural highlights

**Weather Analysis:**
- Daily forecast summary
- Activity recommendations based on weather
- What to pack suggestions

**Day-by-Day Itinerary:**

**Day 1 - <Date>:**
- **Morning (9:00-12:00):** <Activity> - <Reason> - <Budget estimate>
- **Afternoon (12:00-18:00):** <Activity> - <Reason> - <Budget estimate>
- **Evening (18:00-22:00):** <Activity> - <Reason> - <Budget estimate>
- **Weather:** <Forecast for this day>
- **Daily Budget:** $X - $Y

(Repeat for each day)

**Practical Information:**
- **Getting Around:** Local transportation options and tips
- **Where to Stay:** 3 accommodation suggestions (budget/mid-range/luxury)
- **Must-Try Foods:** Local specialties and restaurant types
- **Safety Tips:** Important local considerations
- **Money Matters:** Currency, tipping customs, payment methods
- **Cultural Etiquette:** Do's and don'ts

**Budget Summary:**
- **Total Trip Cost:** $X - $Y (excluding flights)
- **Budget Breakdown:** Accommodation, food, activities, transport

### Rules:
1. Base outdoor/indoor activity recommendations on actual weather data
2. Suggest realistic daily schedules (don't overpack)
3. Provide specific budget ranges in USD
4. Consider seasonal factors (crowds, holidays, closures)
5. Include accessibility info when relevant
6. Be honest about challenges (weather, costs, logistics)

Present information clearly and actionably."""
```

Save and exit.

## Step 4: Register Travel Planner in agents.yaml

Edit:

```bash
nano src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agents.yaml
```

Add this entry under `agents:` list (adjust indentation):

```yaml
- name: "TravelPlannerExecutor"
  type: "executor"
  adapter_class: "ExecutorAdapter"
  description: "Travel planning executor with weather and destination info"

  model:
    provider: "litellm"
    model_id: "openrouter/x-ai/grok-4-fast:free"

  prompt_source: "prompts.executor_prompts.TRAVEL_PLANNER_EXECUTOR_SYSTEM_MESSAGE"

  registration:
    action_keys:
      - action_verb: "execute"
        task_type: "WRITE"
    named_keys:
      - "TravelPlannerExecutor"
      - "travel_planner"

  enabled: true
```

Save and close.

## Step 5: Update Deep Research Agent Profile

Edit profile YAML:

```bash
nano src/sentientresearchagent/hierarchical_agent_framework/agent_configs/profiles/deep_research_agent.yaml
```

Add `"TRAVEL_PLANNING": "TravelPlannerExecutor"` under `executor_adapter_names`:

```yaml
executor_adapter_names:
  SEARCH: "ExaComprehensiveSearcher"
  TRAVEL_PLANNING: "TravelPlannerExecutor"
```

Save.

## Step 6: Rebuild and Relaunch ROMA

```bash
cd ~/ROMA/docker/
sudo docker compose down
sudo docker compose up -d --build
```

Monitor logs:

```bash
sudo docker compose logs -f backend
```

Verify no errors, and TravelPlannerExecutor is loaded.

## Step 7: Test Your Travel Assistant

Access ROMA frontend:

```
http://<your_vps_ip>:3000
```

Try commands like:

- Plan a 5-day trip to Paris
- Create itinerary for Tokyo | March 15-22, 2026
- Suggest a budget week in Barcelona

# (Second Method)
Step 1: Update .env with Travel APIs
Edit your .env file in the ROMA root folder:
bashnano .env
Add these lines at the bottom:
bashOPEN_METEO_BASE_URL=https://api.open-meteo.com/v1/forecast
WIKIPEDIA_API_URL=https://en.wikipedia.org/w/api.php
Save and exit (Ctrl+X, Y, Enter).
Step 2: Add the Travel Planner Adapter
Edit the adapters file:
bashnano src/sentientresearchagent/hierarchical_agent_framework/agents/adapters.py
At the bottom of the file, append this complete class:
pythonclass TravelPlannerAdapter(ExecutorAdapter):
    """Travel planning executor that fetches weather and destination data."""

    def __init__(self, agno_agent_instance=None, agent_name="TravelPlannerAdapter"):
        super().__init__(agno_agent_instance, agent_name)
        self.weather_api = os.getenv("OPEN_METEO_BASE_URL", "https://api.open-meteo.com/v1/forecast")
        self.wiki_api = os.getenv("WIKIPEDIA_API_URL", "https://en.wikipedia.org/w/api.php")

    async def process(self, node, agent_task_input, trace_manager):
        destination = agent_task_input.current_goal.strip()
        logger.info("=" * 80)
        logger.info(f"🗺️ TravelPlannerAdapter.process() CALLED!")
        logger.info(f"🗺️ Destination input: {destination}")
        logger.info(f"🗺️ Node: {node}")
        logger.info(f"🗺️ Model config: {self.model if hasattr(self, 'model') else 'NO MODEL'}")
        logger.info("=" * 80)

        # Parse destination and dates
        parts = destination.split("|")
        location = parts[0].strip()
        dates = parts[1].strip() if len(parts) > 1 else "next 7 days"

        # Get coordinates from Wikipedia
        wiki_coords = self._get_coordinates(location)
        
        # Get weather forecast
        weather_data = {}
        if wiki_coords and wiki_coords.get('lat') != 0:
            weather_data = self._get_weather(wiki_coords['lat'], wiki_coords['lon'])

        # Get destination info
        destination_info = self._get_destination_info(location)

        # Format data for LLM
        enriched_goal = f"""Create a comprehensive travel itinerary for {location} ({dates}).

DESTINATION INFORMATION:
{destination_info}

WEATHER FORECAST DATA:
{weather_data}

COORDINATES: Latitude {wiki_coords.get('lat', 'N/A')}, Longitude {wiki_coords.get('lon', 'N/A')}

Please create a detailed day-by-day itinerary following the travel planning format with weather considerations, budget estimates, and practical tips.
"""
        
        # Override the goal with enriched data
        agent_task_input.current_goal = enriched_goal
        
        # CRITICAL: Force the AgnoAgent to use our configured model
        if hasattr(self, 'agno_agent') and self.agno_agent and hasattr(self, 'model'):
            model_id = self.model.get('model_id') if isinstance(self.model, dict) else None
            if model_id:
                logger.info(f"🚀 Forcing LLM model to: {model_id}")
                # Update the agent's model before calling
                if hasattr(self.agno_agent, 'model'):
                    from agno.models.litellm import LiteLLM
                    self.agno_agent.model = LiteLLM(id=model_id)
                    logger.info(f"✅ Successfully updated AgnoAgent model to {model_id}")
        
        # Call parent ExecutorAdapter's process with the updated model
        return await super().process(node, agent_task_input, trace_manager)

    def _get_coordinates(self, location):
        """Get coordinates from Wikipedia."""
        try:
            params = {
                "action": "query",
                "format": "json",
                "titles": location,
                "prop": "coordinates"
            }
            resp = requests.get(self.wiki_api, params=params, timeout=10)
            data = resp.json()
            pages = data.get("query", {}).get("pages", {})
            for page in pages.values():
                coords = page.get("coordinates")
                if coords:
                    return {"lat": coords[0]["lat"], "lon": coords[0]["lon"]}
        except Exception as e:
            logger.error(f"[Wikipedia Coords] Error: {e}")
        return {"lat": 0, "lon": 0}

    def _get_weather(self, lat, lon):
        """Get 7-day weather forecast from Open-Meteo."""
        try:
            params = {
                "latitude": lat,
                "longitude": lon,
                "daily": "temperature_2m_max,temperature_2m_min,precipitation_sum,weathercode",
                "timezone": "auto"
            }
            resp = requests.get(self.weather_api, params=params, timeout=10)
            resp.raise_for_status()
            return resp.json()
        except Exception as e:
            logger.error(f"[Open-Meteo] Error: {e}")
            return {}

    def _get_destination_info(self, location):
        """Get destination summary from Wikipedia."""
        try:
            params = {
                "action": "query",
                "format": "json",
                "titles": location,
                "prop": "extracts",
                "exintro": True,
                "explaintext": True
            }
            resp = requests.get(self.wiki_api, params=params, timeout=10)
            data = resp.json()
            pages = data.get("query", {}).get("pages", {})
            for page in pages.values():
                return page.get("extract", "No information available")
        except Exception as e:
            logger.error(f"[Wikipedia Info] Error: {e}")
            return "Information unavailable"
Save and exit.
Step 3: Add Executor Prompt
Edit the executor prompts file:
bashnano src/sentientresearchagent/hierarchical_agent_framework/agent_configs/prompts/executor_prompts.py
At the bottom, add this prompt:
pythonTRAVEL_PLANNER_EXECUTOR_SYSTEM_MESSAGE = """You are an expert travel planning assistant specialized in creating comprehensive, personalized itineraries.

### Your Task:
Analyze the provided destination data (weather, attractions, local info) and create a detailed travel itinerary.

### Provided Data Includes:
- Destination name and coordinates
- 7-day weather forecast (temperature, precipitation, conditions)
- Wikipedia destination overview
- Travel dates

### Create a Structured Response:

**Destination Overview:**
- Brief introduction to the location
- Best time to visit considerations
- Cultural highlights

**Weather Analysis:**
- Daily forecast summary
- Activity recommendations based on weather
- What to pack suggestions

**Day-by-Day Itinerary:**

**Day 1 - <Date>:**
- **Morning (9:00-12:00):** <Activity> - <Reason> - <Budget estimate>
- **Afternoon (12:00-18:00):** <Activity> - <Reason> - <Budget estimate>
- **Evening (18:00-22:00):** <Activity> - <Reason> - <Budget estimate>
- **Weather:** <Forecast for this day>
- **Daily Budget:** $X - $Y

(Repeat for each day)

**Practical Information:**
- **Getting Around:** Local transportation options and tips
- **Where to Stay:** 3 accommodation suggestions (budget/mid-range/luxury)
- **Must-Try Foods:** Local specialties and restaurant types
- **Safety Tips:** Important local considerations
- **Money Matters:** Currency, tipping customs, payment methods
- **Cultural Etiquette:** Do's and don'ts

**Budget Summary:**
- **Total Trip Cost:** $X - $Y (excluding flights)
- **Budget Breakdown:** Accommodation, food, activities, transport

### Rules:
1. Base outdoor/indoor activity recommendations on actual weather data
2. Suggest realistic daily schedules (don't overpack)
3. Provide specific budget ranges in USD
4. Consider seasonal factors (crowds, holidays, closures)
5. Include accessibility info when relevant
6. Be honest about challenges (weather, costs, logistics)

Present information clearly and actionably."""
Save and exit.
Step 4: Register TravelPlannerAdapter in Validation Schema
Edit the models file:
bashnano /root/ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/models.py
Find the validate_adapter_class function and update two locations:
Location 1: Add to valid_adapters list
Change FROM:
pythonvalid_adapters = [
    "PlannerAdapter", "ExecutorAdapter", "AtomizerAdapter",
    "AggregatorAdapter", "PlanModifierAdapter",
    "OpenAICustomSearchAdapter", "GeminiCustomSearchAdapter",
    "ExaCustomSearchAdapter"
]
Change TO:
pythonvalid_adapters = [
    "PlannerAdapter", "ExecutorAdapter", "AtomizerAdapter",
    "AggregatorAdapter", "PlanModifierAdapter",
    "OpenAICustomSearchAdapter", "GeminiCustomSearchAdapter",
    "ExaCustomSearchAdapter", "TravelPlannerAdapter"
]
Location 2: Add to type_adapter_map
Change FROM:
pythontype_adapter_map = {
    "planner": ["PlannerAdapter"],
    "executor": ["ExecutorAdapter"],
    "aggregator": ["AggregatorAdapter"],
    "atomizer": ["AtomizerAdapter"],
    "plan_modifier": ["PlanModifierAdapter"],
    "custom_search": ["OpenAICustomSearchAdapter", "GeminiCustomSearchAdapter", "ExaCustomSearchAdapter"]
}
Change TO:
pythontype_adapter_map = {
    "planner": ["PlannerAdapter"],
    "executor": ["ExecutorAdapter", "TravelPlannerAdapter"],
    "aggregator": ["AggregatorAdapter"],
    "atomizer": ["AtomizerAdapter"],
    "plan_modifier": ["PlanModifierAdapter"],
    "custom_search": ["OpenAICustomSearchAdapter", "GeminiCustomSearchAdapter", "ExaCustomSearchAdapter"]
}
Save and exit.
Step 5: Register TravelPlannerAdapter in Agent Factory
Edit the agent factory file:
bashnano /root/ROMA/src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agent_factory.py
Update 1: Add Import (around line 45)
Find this line:
pythonfrom ..agents.adapters import PlannerAdapter, ExecutorAdapter, AtomizerAdapter, AggregatorAdapter, PlanModifierAdapter
Change TO:
pythonfrom ..agents.adapters import (
    PlannerAdapter, 
    ExecutorAdapter, 
    AtomizerAdapter, 
    AggregatorAdapter, 
    PlanModifierAdapter,
    TravelPlannerAdapter
)
Update 2: Add to adapter_classes dictionary (around line 74)
Find:
pythonself.adapter_classes = {
    'PlannerAdapter': PlannerAdapter,
    'ExecutorAdapter': ExecutorAdapter,
    'AtomizerAdapter': AtomizerAdapter,
    'AggregatorAdapter': AggregatorAdapter,
    'PlanModifierAdapter': PlanModifierAdapter,
    'OpenAICustomSearchAdapter': OpenAICustomSearchAdapter,
    'GeminiCustomSearchAdapter': GeminiCustomSearchAdapter,
    'ExaCustomSearchAdapter': ExaCustomSearchAdapter,
}
Change TO:
pythonself.adapter_classes = {
    'PlannerAdapter': PlannerAdapter,
    'ExecutorAdapter': ExecutorAdapter,
    'AtomizerAdapter': AtomizerAdapter,
    'AggregatorAdapter': AggregatorAdapter,
    'PlanModifierAdapter': PlanModifierAdapter,
    'OpenAICustomSearchAdapter': OpenAICustomSearchAdapter,
    'GeminiCustomSearchAdapter': GeminiCustomSearchAdapter,
    'ExaCustomSearchAdapter': ExaCustomSearchAdapter,
    'TravelPlannerAdapter': TravelPlannerAdapter,
}
Save and exit.
Step 6: Register Travel Planner in agents.yaml
Edit:
bashnano src/sentientresearchagent/hierarchical_agent_framework/agent_configs/agents.yaml
Add this entry under the agents: list:
yaml- name: "TravelPlannerExecutor"
  type: "executor"
  adapter_class: "TravelPlannerAdapter"
  description: "Travel planning executor with weather and destination info"

  model:
    provider: "litellm"
    model_id: "openrouter/microsoft/mai-ds-r1:free"

  prompt_source: "prompts.executor_prompts.TRAVEL_PLANNER_EXECUTOR_SYSTEM_MESSAGE"

  registration:
    action_keys:
      - action_verb: "execute"
        task_type: "WRITE"
    named_keys:
      - "TravelPlannerExecutor"
      - "travel_planner"

  enabled: true
Important Notes:

You can change model_id to any free model like "openrouter/x-ai/grok-4-fast:free"
The adapter_class MUST be "TravelPlannerAdapter" (not "ExecutorAdapter")

Save and exit.
Step 7: Disable Conflicting Executors
In the same agents.yaml file, find these agents and disable them to prevent conflicts:
Disable BasicReportWriter
Find:
yaml- name: BasicReportWriter
  ...
  enabled: true
Change to:
yaml  enabled: false
Disable CryptoResearchExecutor
Find:
yaml- name: CryptoResearchExecutor
  ...
  enabled: true
Change to:
yaml  enabled: false
Save and exit.
Step 8: Rebuild and Relaunch ROMA
bashcd ~/ROMA/docker/
sudo docker compose down
sudo docker compose up -d --build
Wait 2-3 minutes for containers to fully start.
Step 9: Verify Startup Logs
Monitor logs to confirm successful registration:
bashsudo docker compose logs -f backend | grep -E "(TravelPlanner|validation error|Overwriting.*WRITE)"
You should see:
✅ Created valid BaseAdapter for TravelPlannerExecutor: TravelPlannerAdapter
AgentRegistry: Registered adapter 'TravelPlannerExecutor' for action 'execute', task_type 'WRITE'
You should NOT see:
Overwriting agent 'TravelPlannerAdapter' with ...
If you see overwriting, another WRITE executor is still enabled—go back and disable it.

<img width="1833" height="834" alt="image" src="https://github.com/user-attachments/assets/65a4a98b-7eea-4281-939d-f9740a3b57cc" />
<img width="1126" height="1200" alt="image" src="https://github.com/user-attachments/assets/d8e15b46-cef4-4787-a4d8-cc7f8a36d180" />



## Summary

- This tutorial builds on ROMA installed with the kasnadoona5/ROMA_sentient free-API optimized setup.
- The Travel Planner Executor uses only free, public data sources for detailed travel plans.
- Use "ExecutorAdapter" and "WRITE" task type as per ROMA expected config.
- The prompt fully guides the agent to generate comprehensive, structured itineraries.
- Ready to run on your VPS with Docker deployment.

Contact me if you want sample adapter code files or have any questions!

Happy travels with ROMA! 🚀🌍
