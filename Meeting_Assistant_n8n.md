##Workflow Overview
This n8n workflow is designed to collect meeting notes and transcripts from Granola app , process them to extract actionable items, and send a daily summary via email.

##Workflow Nodes
Schedule Trigger

###Type: Trigger
Frequency: Daily at midnight
Purpose: Initiates the workflow daily to ensure the processing of the day's meetings.

##Code Node For Integrating Granola MCP/API

###Type: Code Execution
Purpose: Generates the start and end times for the current day in ISO 8601 format.
Output: { start: <start-of-day-time>, end: <end-of-day-time> }

##HTTP Request

###Type: HTTP Request
Method: GET
URL: https://public-api.granola.ai/v1/notes
Authentication: Bearer Token
Purpose: Retrieves notes created within the start and end times specified.

##Loop Over Items

###Type: Batch Processing
Purpose: Processes each meeting note individually.

##Message a Model

##Type: OpenAI Model Interaction
Model: gpt-4o-mini
Purpose: Processes each meeting transcript to extract action items with OpenAI.

##Merge

###Type: Merge
Purpose: Gathers and processes transcripts and their corresponding action items together.

##Code Node For Integrating Granola MCP/API

###Type: Code Execution
Purpose: Formats the processed meeting details into a summary including the action items and sends them in a structured format.

##Send a Message

###Type: Gmail Email
Send To: example@gmail.com
Purpose: Sends the daily meeting summary email with the subject line that indicates it's for the current date.

##Credentials
Requires Bearer token for granola.ai API access.
Requires OpenAI API credentials.
Requires Gmail OAuth2 credentials for sending emails.
