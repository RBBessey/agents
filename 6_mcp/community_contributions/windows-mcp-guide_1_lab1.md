# MCP on Windows: Debugging Guide for Ed Donner's Agents Course

## Problem Statement
Windows users face specific challenges when running MCP (Model Context Protocol) servers in Ed Donner's Agents Course. These challenges include asyncio compatibility issues, event loop conflicts, and standard I/O handling problems that cause the notebook to hang indefinitely.

## Problem Summary
When following the `1_lab1.ipynb` notebook on Windows, users typically encounter:
1. Direct MCP server connections that hang indefinitely
2. Asyncio event loop errors specific to Windows
3. Problems with standard I/O handling between Jupyter and MCP servers

## Solution Overview
To solve these issues, we'll create a specialized script that runs MCP servers in separate processes, addressing Windows-specific asyncio challenges.

## Step-by-Step Guide

### 1. Create the Scripts Directory
First, create a directory to store our helper scripts:

```python
import os
os.makedirs(os.path.join(os.getcwd(), "scripts"), exist_ok=True)
```

### 2. Create the run_agent.py Script
Create a file named `run_agent.py` in the scripts directory with the following code:

```python
# run_agent.py
import asyncio
import sys
import os
from dotenv import load_dotenv
from agents import Agent, Runner, trace
from agents.mcp import MCPServerStdio

async def main():
    load_dotenv(override=True)
    
    # Explicit setup of ProactorEventLoop for Windows
    if sys.platform == 'win32':
        asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
    
    # Agent instructions
    instructions = """
    You browse the internet to accomplish your instructions.
    You are highly capable at browsing the internet independently to accomplish your task, 
    including accepting all cookies and clicking 'not now' as
    appropriate to get to the content you need. If one website isn't fruitful, try another. 
    Be persistent until you have solved your assignment,
    trying different options and sites as needed.
    """
    
    # Make sure the sandbox directory exists
    sandbox_path = os.path.abspath(os.path.join(os.getcwd(), "sandbox"))
    os.makedirs(sandbox_path, exist_ok=True)
    
    # Configure MCP servers
    fetch_params = {"command": "uvx", "args": ["mcp-server-fetch"]}
    files_params = {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", sandbox_path]}
    playwright_params = {"command": "npx", "args": ["@playwright/mcp@latest"]}
    
    # Run the agent with all three MCP servers
    try:
        async with MCPServerStdio(params=files_params, cache_tools_list=True) as mcp_server_files:
            async with MCPServerStdio(params=playwright_params, cache_tools_list=True) as mcp_server_browser:
                # Start with just two servers that we know are working
                agent = Agent(
                    name="investigator", 
                    instructions=instructions, 
                    model="gpt-4o-mini",
                    mcp_servers=[mcp_server_files, mcp_server_browser]
                )
                with trace("investigate"):
                    result = await Runner.run(agent, "Find a great recipe for Banoffee Pie, then summarize it in markdown to banoffee.md")
                    print("AGENT RESULT:", result.final_output)
    except Exception as e:
        print(f"Error running agent: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 3. Prerequisites Installation
Install the required packages:

```bash
# Activate the virtual environment
c:\Users\[Username]\Documents\projects\agents\.venv\Scripts\activate

# Install Python dependencies
pip install python-dotenv

# Install Node.js dependencies (if needed)
npm install -g @modelcontextprotocol/server-filesystem
npm install -g @playwright/mcp
```

### 4. Run the Agent from Jupyter
In your Jupyter notebook (`1_lab1.ipynb`), add a cell with the following code to execute the script:

```python
import os
import subprocess
import sys
import time

# Get paths
current_dir = os.getcwd()
parent_dir = os.path.dirname(current_dir)
script_abs_path = os.path.join(current_dir, "scripts", "run_agent.py")
python_path = os.path.join(parent_dir, ".venv", "Scripts", "python.exe")

# Print the command we're running
command = [python_path, script_abs_path]
print(f"Running agent with command: {' '.join(command)}")

# Start the process with real-time monitoring
process = subprocess.Popen(
    command,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    bufsize=1  # Line buffered
)

# Stream output in real-time
start_time = time.time()
timeout = 300  # 5 minutes timeout for the agent to complete its task

while process.poll() is None:  # While process is still running
    # Check for timeout
    if time.time() - start_time > timeout:
        print(f"Process took longer than {timeout} seconds. Terminating...")
        process.terminate()
        break
        
    # Check for output
    if process.stdout.readable():
        line = process.stdout.readline()
        if line:
            print(f"OUT: {line.rstrip()}")
    
    if process.stderr.readable():
        line = process.stderr.readline()
        if line:
            print(f"ERR: {line.rstrip()}")
    
    time.sleep(0.1)  # Prevent CPU spinning

# Get any remaining output
remaining_stdout, remaining_stderr = process.communicate()
if remaining_stdout:
    print(f"Remaining stdout: {remaining_stdout}")
if remaining_stderr:
    print(f"Remaining stderr: {remaining_stderr}")

print(f"Process finished with return code: {process.returncode}")
```

### 5. Expected Output
When the script runs successfully, you'll see output like:

```
OUT: AGENT RESULT: [The Banoffee Pie recipe summary]
```

You might also see error messages like "Exception ignored in: <function BaseSubprocessTransport.\__del\__ at 0x...>" - these are normal cleanup errors in Windows asyncio and don't affect functionality.

### 6. Saving the Recipe
If the agent doesn't automatically save the recipe (which is a common issue), you can manually save it with:

```python
import os

# Make sure the sandbox directory exists
sandbox_path = os.path.join(os.getcwd(), "sandbox")
os.makedirs(sandbox_path, exist_ok=True)

# The recipe markdown content
recipe = """# Banoffee Pie Recipe
[Copy the recipe from the agent output here]
"""

# Write the recipe to banoffee.md in the sandbox directory
banoffee_path = os.path.join(sandbox_path, "banoffee.md")
with open(banoffee_path, 'w') as f:
    f.write(recipe)

print(f"Recipe saved to {banoffee_path}")
```

## Common Issues and Solutions

1. **NotImplementedError in asyncio**: This is a Windows-specific issue with asyncio's subprocess handling. Our approach of using a separate script with the correct event loop policy works around this.

2. **MCP server connection issues**: If you see errors connecting to MCP servers, verify that Node.js is installed and that you've installed the required global packages.

3. **Script times out but shows results**: If you see the recipe in the output but also see timeout messages, the script actually worked! The agent is generating output but might take longer than expected.

4. **The agent doesn't save files**: If the recipe isn't saved to disk, use the manual file saving code above to save it yourself.

By following this guide, Windows users can successfully run the MCP components of Ed Donner's Agents Course, working around the Windows-specific asyncio and subprocess handling issues.
