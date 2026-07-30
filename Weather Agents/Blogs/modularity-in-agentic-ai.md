# Modular Tool Design for Optimal Performance of Autonomous Agents

## Modularity as a design principle is a critical enabler for Agentic AI systems. This holds true regardless of the problem being solved and whether it is a single agent or a multi-agent set-up.  In this blog i use a simple single agent exercise to explore how modularity in tool design enables an Agentic AI solution to extend and adapt to future use-cases.

### Here is a Monolithic design example that works but scales poorly. 
```python
### @tool
def get_weather(city: str) -> str:
    """Get 5-day forecast for a given city in a single call. Return the daily max temperature and issue a travel
    advisory (Do not Travel/Safe to Travel).
    """
   
    try:
        api_call_params = {"q":city, "appid":api_key, "units":"metric"}
        Api_Response = requests.get(url,params=api_call_params)
        
        if Api_Response.status_code ==200:
            data = Api_Response.json()
            weather_forecasts = {}
            for entry in data["list"]:
                date = entry["dt_txt"].split()[0]
                temp = entry["main"]["temp"]
                weather_forecasts.setdefault(date,[]).append(temp)

            Weather_Advisory = []
            for date, temps in weather_forecasts.items():
                max_temp = max(temps)
                if max_temp > 40:
                    Advice = "Extreme Heat : Do not Travel"

                else:
                    Advice = "Safe to Travel"

                Weather_Advisory.append(f"{date}: {max_temp:.1f}.{Advice}")

            return ("\n".join(Weather_Advisory)+"\n")
            
            
        else:
            print(f"API call failed with status {Api_Response.status_code}")

    except Exception as e:
        return (f"Error occurred: {str(e)}")
```
### What makes this function design less than an optimal choice? 

The function's length is an early warning sign – it's trying to do too much. 

- Monolithic: API Call, parsing, logic and formatting are all boxed into a single function.
- No separation of concerns: Any extension requires a major re-factor. As requirements grow, the function becomes harder to maintain.
- Hard to test: Unit testing becomes difficult. You can’t just test the advisory logic without executing the whole function.
- Poor traceability: Debugging is cumbersome.
- Understanding Gaps: Long functions run the risk of being poorly understood by the LLM’s. It must unpack everything before suggesting fixes or improvements.
- Potentially increased cognitive load increases the risk of hallucinated responses by the LLM.

### Modularity in tool design means that the functions are lucid and clear for the LLM's. A critical consequence of this is that the functions are self-contained. There are extremely clear and unambiguous descriptions explaining the goal of the function, its input arguments, and its output.

### Modular Design example: 

###
```python
def fetch_weather_data(city: str) -> dict:

    """Get a 5-day forecast for a city in a single call. 
      Args: 
      city: Name of the city passed as a string
      output: A dictionary with raw weather data.
    """   
    API_Call_Params = {"q":city, "appid":api_key, "units":"metric"} 

    API_Response = requests.get(url,params=API_Call_Params)  

    return API_Response


def process_weather_data(data:dict) -> dict:
    """Converts raw weather data into a structured json.
    Parse the json to extract temperatures for each date.
    Returns a dictionary with forecasted temperatures for next 5 days.
    """ 

    if data.status_code == 200:
        API_Response_json = data.json()
    else:
        print ("API called with status : {API_Response.status_code}")

    weather_forecasts = {}

    for entry in API_Response_json["list"]:
        date = entry["dt_txt"].split()[0]
        temp = entry["main"]["temp"]
        weather_forecasts.setdefault(date,[]).append(temp)

    return weather_forecasts


def weather_advisory(weather_forecasts : dict) -> list:
    """Generate travel advisory after checking the maximum temperature.
    Format the output string.
    Return a list containing the date, maximum temperature and the advisory.
""" 
    advisory = []
    for date, temps in weather_forecasts.items():
        max_temp = max(temps)
        if max_temp > 40:
            advice = "Extreme Heat: Do not Travel"
        else:
            advice = "Safe to Travel"
            advisory.append(f"{date}: {max_temp:.1f}.{advice}") 

    return advisory


@tool # To register the python function as an agentic tool. Returns a tool object.
def get_weather(city : str) -> str:
    """Call and execute all the helper functions.
    Args:
    City : The name of the city, for which forecast is requested, passed as a str.
    Output: A multi-line string of advisories.
    """
    #Tools pipeline orchestration.    
    data = fetch_weather_data(city)
    forecasts = process_weather_data(data)
    advisories = weather_advisory(forecasts)
    
    return "\n".join (advisories)

```

### The above modular organization of functions is easy to read and provides a clear understanding of what each function is doing. There is a progressive build-up of the solution between one function to the next, as it moves from raw data to a travel advisory.

**If a human reader can definitely say what each function does, one can rest assured that an AI agent will understand it well.**

Modularity has allowed us to enhance each function in a way that will allow an AI agent to accurately identify the right tool/function to achieve a given task. 

- Separation of concerns: Each function has a single clear responsibility.
- Docstrings: Aligned and specific docstrings all the AI agent to map tasks to the correct function.
- Enhancements: Enhancing your advisories to include "feels like", "humidity", "visibility" are great value-add's. New feature additions don't require touching all the functions.
- Testing:  Each function can be tested independently with it's own edge cases.
- Scaling - Transitioning to a multi-agent set-up is easier where each agent can own a specific function.  
 
## Why is modularity in tool design crucial for the success of Autonomous Agents?

### Tool selection:
Agents rely on tools to retrieve current information; a crucial ability that allows them to accomplish goals. Tools give agents the agency to autonomously complete tasks.  Well written crisp functions with clear responsibility allow the agents to select the right tool to accomplish a given task.

### Scaling:
Modular functions articulated with a clear responsibility make it easier to transition from a single agent to a multi-agent set-up, where each agent can be linked to a specific tool/function.

### Parallelism:
In a multi-agent set-up functions and tools can be executed parallely as they have minimal overlap amongst themselves. A modular tool design with clear separation of concerns enables parallel execution of functions.



## Agentic Execution Framework

| Agentic Framework        | Tools/Functions                                                                 | Purpose                                                                 | Agent Linkage                                                                                      | Notes                                      |
|--------------------------|---------------------------------------------------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|--------------------------------------------|
| Single Agent (Monolithic)| `get_weather()`                                                                 | One bloated function doing API call, parsing, advisory logic, formatting | `agent = tools[get_weather]`                                                                       | Bloated design, hard to scale.             |
| Single Agent (Modular)   | `fetch_weather_data()`<br>`process_weather_data()`<br>`weather_advisory()`<br>`get_weather()` | Clear separation of concerns, modular execution                         | `agent = tools[get_weather]`                                                                       | Executor agent runs `get_weather()`, which sequentially invokes modular tools. Easier to extend. |
| Multi Agent Execution    | `fetch_weather_data()`<br>`process_weather_data()`<br>`weather_advisory()`       | Each tool independently executed by its agent                           | `Data_Agent = tools[fetch_weather_data]`<br>`Transform_Agent = tools[process_weather_data]`<br>`Advisory_Agent = tools[weather_advisory]` | Each agent executes independently. Coordination handled by `Coordinator_Agent`. |
| Coordinator Agent        | `get_weather()`                                                                 | Orchestrates modular tools                                              | `Coordinator_Agent = tools[get_weather]`                                                           | Parallel execution possible.                |





 
   
   
    	     
