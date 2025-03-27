# Life Logs Reminder

An automatic integration that extracts events and tasks from Limitless Life Logs and adds them to Google Calendar and Google Tasks.

![Life Logs to Calendar & Tasks](https://via.placeholder.com/800x400?text=Life+Logs+Integration)

## Overview

This project automatically:

1. Checks your Limitless Life Logs for mentions of appointments, events, and tasks
2. Uses AI to extract details like dates, times, and contextual information
3. Adds appointments to your Google Calendar
4. Adds reminders/tasks to Google Tasks
5. Sends you email notifications when new items are found

It runs automatically every few minutes to keep your calendar and task list up-to-date with things mentioned in your conversations.

## Prerequisites

To use this integration, you'll need:

- A [Limitless](https://limitless.ai) account with API access
- An [OpenAI API key](https://platform.openai.com/api-keys)
- A Google account (for Google Apps Script, Calendar, and Tasks)

## Setup Instructions

### Step 1: Get Your API Keys

1. **Limitless API Key**:
   - Go to your Limitless account settings
   - Find or generate your API key

2. **OpenAI API Key**:
   - Go to [OpenAI API Keys](https://platform.openai.com/api-keys)
   - Create a new API key (if you don't have one already)
   - Copy the key (you'll need it for the script)

### Step 2: Create a Google Apps Script Project

1. Go to [Google Apps Script](https://script.google.com)
2. Click "New Project"
3. Name your project (e.g., "Life Logs Integration")
4. Delete any default code in the editor

### Step 3: Add the Script Code

Copy and paste the entire script code below into the editor:

```javascript
// Configuration - Replace with your actual API keys
const LIMITLESS_API_KEY = "YOUR_LIMITLESS_API_KEY";
const OPENAI_API_KEY = "YOUR_OPENAI_API_KEY"; // Get this from https://platform.openai.com/api-keys

// How many hours back to check for new logs
const HOURS_LOOKBACK = 24;

// Calendar settings
const CALENDAR_NAME = "Life Logs Appointments"; // For appointments only
const DEFAULT_REMINDER_MINUTES = 30; // Default reminder time in minutes

// Main function that runs the entire process
function processLifeLogs() {
  console.log("Starting Life Logs processing...");
  
  // Step 1: Get recent life logs
  const recentLogs = getRecentLifeLogs();
  if (!recentLogs || recentLogs.length === 0) {
    console.log("No recent logs found.");
    return;
  }
  
  console.log(`Found ${recentLogs.length} recent logs.`);
  
  // Step 2: Process logs with OpenAI to extract events
  for (const log of recentLogs) {
    const logContent = log.markdown || "";
    if (!logContent.trim()) {
      console.log(`Log ${log.id} has no content, skipping.`);
      continue;
    }
    
    console.log(`Processing log: "${log.title}"`);
    const events = extractEventsWithOpenAI(logContent);
    
    // Step 3: Process events (appointments to calendar, reminders to tasks)
    if (events && events.length > 0) {
      console.log(`Found ${events.length} events/tasks in log.`);
      processEvents(events, log);
    } else {
      console.log("No events or tasks found in this log.");
    }
  }
  
  console.log("Life Logs processing completed.");
}

// Get recent life logs from the Limitless API
function getRecentLifeLogs() {
  const now = new Date();
  const startTime = new Date(now.getTime() - (HOURS_LOOKBACK * 60 * 60 * 1000));
  
  const options = {
    method: 'GET',
    headers: {
      'X-API-Key': LIMITLESS_API_KEY
    },
    muteHttpExceptions: true
  };
  
  try {
    // Get logs from the last HOURS_LOOKBACK hours
    const response = UrlFetchApp.fetch(`https://api.limitless.ai/v1/lifelogs?limit=20`, options);
    const responseData = JSON.parse(response.getContentText());
    
    if (responseData && responseData.data && responseData.data.lifelogs) {
      // Filter logs that are newer than our lookback time
      return responseData.data.lifelogs.filter(log => {
        const logDate = new Date(log.startTime);
        return logDate >= startTime;
      });
    }
    return [];
  } catch (error) {
    console.error("Error fetching life logs:", error);
    return [];
  }
}

// Use OpenAI to extract events and reminders from the log content
function extractEventsWithOpenAI(logContent) {
  const prompt = `
You are an AI assistant specialized in extracting calendar events and reminders from conversation transcripts.

INSTRUCTIONS:
1. Read the transcript carefully
2. Identify ANY mentions of:
   - Appointments (doctor, dental, etc.)
   - Scheduled meetings
   - Deadlines
   - Tasks/reminders (like buying something)
   - Social events
   - Any dates or times mentioned

3. For EACH event or reminder identified, create a JSON object with:
   - title: Clear, concise title
   - date: YYYY-MM-DD format OR "today"/"tomorrow"
   - time: HH:MM format, or leave empty if no time
   - isReminder: true for tasks/reminders, false for appointments/events
   - details: Context and additional information

SPECIAL RULES:
1. If you find an appointment but NO clear date, mark it as a reminder (isReminder: true) and set the title to "Follow up about: [appointment name]"
2. Always set "today" for the date if something needs to be done immediately (like "Buy milk")
3. If there's no specific deadline mentioned for a task, assume it's for today

BE VERY THOROUGH - any mention of a future event or task should be captured.

OUTPUT FORMAT:
Respond ONLY with a valid JSON array of events. If no events/reminders are found, return an empty array [].

Example response:
[
  {
    "title": "Dental Checkup",
    "date": "2023-11-14",
    "time": "15:30",
    "isReminder": false,
    "details": "Scheduled dental appointment"
  },
  {
    "title": "Buy milk",
    "date": "today",
    "time": "",
    "isReminder": true,
    "details": "Need to get milk from the store"
  },
  {
    "title": "Follow up about: Eye Exam",
    "date": "today",
    "time": "",
    "isReminder": true,
    "details": "Need to confirm date and time for upcoming eye exam"
  }
]

TRANSCRIPT TO ANALYZE:
${logContent}

IMPORTANT: Based on this transcript, what events or reminders should be added to a calendar? Respond with a JSON array only.
`;

  const options = {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${OPENAI_API_KEY}`
    },
    payload: JSON.stringify({
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "system",
          content: "You are a helpful assistant that extracts calendar events and reminders from text. Only respond with valid JSON."
        },
        {
          role: "user",
          content: prompt
        }
      ],
      temperature: 0.1,
      max_tokens: 2048
    }),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
    const responseCode = response.getResponseCode();
    console.log(`OpenAI API response code: ${responseCode}`);
    
    const responseText = response.getContentText();
    console.log("OpenAI response received. Length: " + responseText.length);
    
    // Check for API errors
    if (responseCode !== 200) {
      console.error("OpenAI API Error:", responseText);
      return [];
    }
    
    const responseData = JSON.parse(responseText);
    
    if (responseData && responseData.choices && responseData.choices.length > 0) {
      const text = responseData.choices[0].message.content;
      console.log("Raw OpenAI response:", text);
      
      try {
        // Try to parse the JSON directly
        const parsedEvents = JSON.parse(text);
        console.log(`Successfully parsed ${parsedEvents.length} events`);
        return parsedEvents;
      } catch (e) {
        // If direct parsing fails, try to extract JSON from the text
        const jsonMatch = text.match(/\[\s*\{.*\}\s*\]/s);
        if (jsonMatch) {
          console.log("JSON pattern found in response");
          try {
            const parsedEvents = JSON.parse(jsonMatch[0]);
            console.log(`Successfully parsed ${parsedEvents.length} events from pattern`);
            return parsedEvents;
          } catch (e2) {
            console.error("Failed to parse JSON from pattern:", e2);
            return [];
          }
        } else {
          console.log("No JSON array found in OpenAI response");
          return [];
        }
      }
    }
    
    return [];
  } catch (error) {
    console.error("Error calling OpenAI API:", error);
    console.error("Error details:", error.toString());
    return [];
  }
}

// Process events - add appointments to calendar and reminders to tasks
function processEvents(events, log) {
  let appointmentsAdded = 0;
  let tasksAdded = 0;
  
  // Keep track of newly added events for notification
  const addedEvents = [];
  
  for (const event of events) {
    try {
      if (event.isReminder) {
        // Add to Google Tasks
        if (addToGoogleTasks(event, log)) {
          tasksAdded++;
          addedEvents.push(event);
        }
      } else {
        // Add to Google Calendar
        if (addToGoogleCalendar(event, log)) {
          appointmentsAdded++;
          addedEvents.push(event);
        }
      }
    } catch (error) {
      console.error(`Error processing event "${event.title}":`, error);
    }
  }
  
  console.log(`Added ${appointmentsAdded} appointments to calendar and ${tasksAdded} tasks to Google Tasks.`);
  
  // Send email notification if anything was added
  if (addedEvents.length > 0) {
    sendEmailNotification(addedEvents, log);
  }
}

// Add an appointment to Google Calendar
function addToGoogleCalendar(event, log) {
  try {
    // Parse the date and time
    const eventDateTime = parseDateTime(event.date, event.time);
    if (!eventDateTime) {
      console.log(`Could not parse date/time for event: ${event.title}`);
      return false;
    }
    
    // Get or create the calendar
    const calendar = getOrCreateCalendar(CALENDAR_NAME);
    
    // Create event details
    const title = event.title;
    const details = `${event.details || ''}\n\nExtracted from Life Log: "${log.title}"\nLog date: ${new Date(log.startTime).toLocaleString()}`;
    
    // Check if this event already exists to avoid duplicates
    if (eventAlreadyExists(calendar, title, eventDateTime)) {
      console.log(`Event "${title}" already exists in calendar, skipping.`);
      return false;
    }
    
    // Debug logging of dates
    console.log(`Creating calendar event: ${title} on ${eventDateTime.start.toLocaleString()} to ${eventDateTime.end.toLocaleString()}`);
    
    // Create the calendar event
    const calendarEvent = calendar.createEvent(
      title,
      eventDateTime.start,
      eventDateTime.end,
      {
        description: details
      }
    );
    
    // Add a reminder
    calendarEvent.addPopupReminder(DEFAULT_REMINDER_MINUTES);
    
    console.log(`Added appointment: "${title}" on ${eventDateTime.start.toLocaleString()}`);
    return true;
  } catch (error) {
    console.error(`Error adding appointment "${event.title}" to calendar:`, error);
    return false;
  }
}

// Add a reminder to Google Tasks
function addToGoogleTasks(event, log) {
  try {
    // Get the default task list
    const taskLists = Tasks.Tasklists.list().items;
    if (!taskLists || taskLists.length === 0) {
      console.error("No task lists found");
      return false;
    }
    
    const taskList = taskLists[0]; // Use the first (default) task list
    
    // Parse the date for the due date
    const dueDate = parseDueDate(event.date, event.time);
    if (!dueDate) {
      console.log(`Could not parse due date for task: ${event.title}`);
      return false;
    }
    
    // Format to RFC 3339 format which Google Tasks API requires
    const dueDateRFC3339 = dueDate.toISOString();
    
    // Create task details
    const title = event.title;
    const notes = `${event.details || ''}\n\nExtracted from Life Log: "${log.title}"\nLog date: ${new Date(log.startTime).toLocaleString()}`;
    
    // Check if this task already exists to avoid duplicates
    if (taskAlreadyExists(taskList.id, title, dueDateRFC3339)) {
      console.log(`Task "${title}" already exists, skipping.`);
      return false;
    }
    
    // Create the task with high priority
    const task = {
      title: "⚠️ " + title, // Add priority indicator in the title
      notes: notes,
      due: dueDateRFC3339,
      status: 'needsAction'
    };
    
    Tasks.Tasks.insert(task, taskList.id);
    
    console.log(`Added high-priority task: "${title}" due on ${dueDate.toLocaleString()}`);
    return true;
  } catch (error) {
    console.error(`Error adding task "${event.title}" to Google Tasks:`, error);
    return false;
  }
}

// Get an existing calendar or create a new one with the given name
function getOrCreateCalendar(calendarName) {
  // Try to find an existing calendar with this name
  const calendars = CalendarApp.getAllCalendars();
  for (const cal of calendars) {
    if (cal.getName() === calendarName) {
      return cal;
    }
  }
  
  // If not found, create a new calendar
  const newCalendar = CalendarApp.createCalendar(calendarName);
  console.log(`Created new calendar: ${calendarName}`);
  return newCalendar;
}

// Check if an event with similar details already exists
function eventAlreadyExists(calendar, title, dateTime) {
  const existingEvents = calendar.getEvents(
    new Date(dateTime.start.getTime() - 60 * 60 * 1000), // Look 1 hour before
    new Date(dateTime.start.getTime() + 60 * 60 * 1000)  // Look 1 hour after
  );
  
  for (const event of existingEvents) {
    if (event.getTitle().toLowerCase().includes(title.toLowerCase())) {
      return true;
    }
  }
  
  return false;
}

// Check if a task already exists (including completed tasks)
function taskAlreadyExists(taskListId, title, dueDate) {
  try {
    // Format the due date to match Google Tasks format (removing milliseconds)
    const formattedDueDate = dueDate.split('.')[0] + 'Z';
    
    // Get all incomplete tasks in the list
    const activeTasks = Tasks.Tasks.list(taskListId, {
      showCompleted: false
    }).items || [];
    
    // Get all completed tasks in the list (from the last 7 days)
    const completedTasks = Tasks.Tasks.list(taskListId, {
      showCompleted: true,
      showHidden: true,
      completedMin: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString()
    }).items || [];
    
    // Combine active and completed tasks
    const allTasks = [...activeTasks, ...completedTasks];
    
    // Check for a task with similar title and due date
    for (const task of allTasks) {
      // Try different matching strategies
      
      // 1. Direct title matching (case insensitive)
      const titleMatch = task.title.toLowerCase().includes(title.toLowerCase()) ||
                        title.toLowerCase().includes(task.title.toLowerCase());
      
      // 2. Remove emoji and other markers for comparison
      const cleanTaskTitle = task.title.replace(/^⚠️\s*/, '');
      const cleanMatch = cleanTaskTitle.toLowerCase().includes(title.toLowerCase()) ||
                         title.toLowerCase().includes(cleanTaskTitle.toLowerCase());
      
      // Check if we have a title match and similar due date
      if ((titleMatch || cleanMatch) && 
          (!task.due || task.due.startsWith(formattedDueDate.substring(0, 10)))) {
        console.log(`Task "${title}" already exists (${task.status}): "${task.title}"`);
        return true;
      }
    }
    
    return false;
  } catch (error) {
    console.error("Error checking existing tasks:", error);
    return false; // If we can't check, assume it doesn't exist
  }
}

// Helper function to parse date and time strings into Date objects for calendar events
function parseDateTime(dateStr, timeStr) {
  try {
    const now = new Date();
    let eventDate;
    
    // Parse the date
    if (dateStr.toLowerCase() === 'today') {
      eventDate = new Date(now);
    } else if (dateStr.toLowerCase() === 'tomorrow') {
      eventDate = new Date(now);
      eventDate.setDate(eventDate.getDate() + 1);
    } else if (dateStr.match(/^\d{4}-\d{2}-\d{2}$/)) {
      // ISO format YYYY-MM-DD
      eventDate = new Date(dateStr + 'T00:00:00');
    } else {
      // Try to parse other formats
      const dateParts = dateStr.split(/[\/\-\.]/);
      if (dateParts.length === 3) {
        // Handle various date formats
        let year, month, day;
        
        // Check if first part looks like a year
        if (dateParts[0].length === 4 && !isNaN(dateParts[0])) {
          // YYYY-MM-DD format
          year = parseInt(dateParts[0]);
          month = parseInt(dateParts[1]) - 1; // Month is 0-based in JS
          day = parseInt(dateParts[2]);
        } else {
          // MM/DD/YYYY format (or similar)
          month = parseInt(dateParts[0]) - 1;
          day = parseInt(dateParts[1]);
          
          // Year might be 2 or 4 digits
          year = parseInt(dateParts[2]);
          if (year < 100) {
            year += 2000; // Assume 20xx for 2-digit years
          }
        }
        
        eventDate = new Date(year, month, day);
      } else {
        // Try natural language parsing
        const monthNames = ["january", "february", "march", "april", "may", "june", "july", "august", "september", "october", "november", "december"];
        const monthAbbr = ["jan", "feb", "mar", "apr", "may", "jun", "jul", "aug", "sep", "oct", "nov", "dec"];
        
        // Check for formats like "14th November" or "November 14th"
        let monthIndex = -1;
        let day = -1;
        
        // Look for month name
        const lowerDateStr = dateStr.toLowerCase();
        for (let i = 0; i < monthNames.length; i++) {
          if (lowerDateStr.includes(monthNames[i]) || lowerDateStr.includes(monthAbbr[i])) {
            monthIndex = i;
            break;
          }
        }
        
        // Look for day number
        const dayMatch = lowerDateStr.match(/\b(\d{1,2})(st|nd|rd|th)?\b/);
        if (dayMatch) {
          day = parseInt(dayMatch[1]);
        }
        
        if (monthIndex >= 0 && day > 0) {
          // We found both month and day
          const year = now.getFullYear(); // Assume current year
          eventDate = new Date(year, monthIndex, day);
          
          // If the date is in the past by more than a few days, assume it's for next year
          if (eventDate < now && now - eventDate > 7 * 24 * 60 * 60 * 1000) {
            eventDate.setFullYear(year + 1);
          }
        } else {
          console.log(`Couldn't parse date: ${dateStr}`);
          return null;
        }
      }
    }
    
    console.log(`Parsed date: ${eventDate.toISOString()} from "${dateStr}"`);
    
    // Parse the time
    if (timeStr && timeStr.trim()) {
      const time = timeStr.trim();
      let hours = 0;
      let minutes = 0;
      
      const timeParts = time.match(/(\d{1,2}):(\d{2})\s*(am|pm)?/i);
      if (timeParts) {
        hours = parseInt(timeParts[1]);
        minutes = parseInt(timeParts[2]);
        
        // Handle AM/PM
        if (timeParts[3] && timeParts[3].toLowerCase() === 'pm' && hours < 12) {
          hours += 12;
        } else if (timeParts[3] && timeParts[3].toLowerCase() === 'am' && hours === 12) {
          hours = 0;
        }
      } else {
        // Try other formats like "3pm"
        const simpleTime = time.match(/(\d{1,2})\s*(am|pm)/i);
        if (simpleTime) {
          hours = parseInt(simpleTime[1]);
          
          // Handle AM/PM
          if (simpleTime[2].toLowerCase() === 'pm' && hours < 12) {
            hours += 12;
          } else if (simpleTime[2].toLowerCase() === 'am' && hours === 12) {
            hours = 0;
          }
        }
      }
      
      eventDate.setHours(hours, minutes, 0, 0);
      console.log(`Added time: ${hours}:${minutes} from "${timeStr}"`);
    } else {
      // Default to 7:00 AM for all-day events as requested
      eventDate.setHours(7, 0, 0, 0);
      console.log(`No time specified, defaulting to 7:00 AM`);
    }
    
    // Create end time (1 hour later for timed events, end of day for all-day)
    const endDate = new Date(eventDate);
    if (timeStr && timeStr.trim()) {
      endDate.setHours(endDate.getHours() + 1);
    } else {
      // For events without specified time, end time is 1 hour after start (8 AM)
      endDate.setHours(8, 0, 0, 0);
    }
    
    return {
      start: eventDate,
      end: endDate
    };
  } catch (error) {
    console.error(`Error parsing date "${dateStr}" and time "${timeStr}":`, error);
    return null;
  }
}

// Parse due date for Google Tasks (simpler than calendar events)
function parseDueDate(dateStr, timeStr) {
  try {
    const now = new Date();
    let dueDate;
    
    // Parse the date
    if (dateStr.toLowerCase() === 'today') {
      // For "today" items, set to the current time
      dueDate = new Date(now);
    } else if (dateStr.toLowerCase() === 'tomorrow') {
      dueDate = new Date(now);
      dueDate.setDate(dueDate.getDate() + 1);
    } else if (dateStr.match(/^\d{4}-\d{2}-\d{2}$/)) {
      // ISO format YYYY-MM-DD
      dueDate = new Date(dateStr + 'T00:00:00');
    } else {
      // Use the same logic as parseDateTime but simpler
      const dateParts = dateStr.split(/[\/\-\.]/);
      if (dateParts.length === 3) {
        let year, month, day;
        
        if (dateParts[0].length === 4 && !isNaN(dateParts[0])) {
          year = parseInt(dateParts[0]);
          month = parseInt(dateParts[1]) - 1;
          day = parseInt(dateParts[2]);
        } else {
          month = parseInt(dateParts[0]) - 1;
          day = parseInt(dateParts[1]);
          year = parseInt(dateParts[2]);
          if (year < 100) year += 2000;
        }
        
        dueDate = new Date(year, month, day);
      } else {
        // Intelligent parsing of date strings
        const monthNames = ["january", "february", "march", "april", "may", "june", "july", "august", "september", "october", "november", "december"];
        const monthAbbr = ["jan", "feb", "mar", "apr", "may", "jun", "jul", "aug", "sep", "oct", "nov", "dec"];
        
        let monthIndex = -1;
        let day = -1;
        
        const lowerDateStr = dateStr.toLowerCase();
        
        // Special case for "next x" expressions
        if (lowerDateStr.includes("next")) {
          const dayOfWeek = ["sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"];
          const dayAbbr = ["sun", "mon", "tue", "wed", "thu", "fri", "sat"];
          
          for (let i = 0; i < dayOfWeek.length; i++) {
            if (lowerDateStr.includes(dayOfWeek[i]) || lowerDateStr.includes(dayAbbr[i])) {
              // Get the next occurrence of that day
              dueDate = new Date(now);
              const currentDay = dueDate.getDay();
              const daysToAdd = (i + 7 - currentDay) % 7;
              dueDate.setDate(dueDate.getDate() + (daysToAdd === 0 ? 7 : daysToAdd));
              break;
            }
          }
          
          // Handle "next week", "next month", etc.
          if (!dueDate) {
            dueDate = new Date(now);
            if (lowerDateStr.includes("week")) {
              dueDate.setDate(dueDate.getDate() + 7);
            } else if (lowerDateStr.includes("month")) {
              dueDate.setMonth(dueDate.getMonth() + 1);
            } else if (lowerDateStr.includes("year")) {
              dueDate.setFullYear(dueDate.getFullYear() + 1);
            }
          }
        } else {
          // Look for month name
          for (let i = 0; i < monthNames.length; i++) {
            if (lowerDateStr.includes(monthNames[i]) || lowerDateStr.includes(monthAbbr[i])) {
              monthIndex = i;
              break;
            }
          }
          
          // Look for day number
          const dayMatch = lowerDateStr.match(/\b(\d{1,2})(st|nd|rd|th)?\b/);
          if (dayMatch) {
            day = parseInt(dayMatch[1]);
          }
          
          if (monthIndex >= 0 && day > 0) {
            const year = now.getFullYear();
            dueDate = new Date(year, monthIndex, day);
            
            // If the date is in the past, assume it's for next year
            if (dueDate < now) {
              dueDate.setFullYear(year + 1);
            }
          }
        }
      }
    }
    
    // If we failed to parse a date, default to today
    if (!dueDate) {
      console.log(`Failed to parse date "${dateStr}", defaulting to today`);
      dueDate = new Date(now);
    }
    
    // Add time if specified (important for notifications)
    if (timeStr && timeStr.trim()) {
      const time = timeStr.trim();
      let hours = 0;
      let minutes = 0;
      
      const timeParts = time.match(/(\d{1,2}):(\d{2})\s*(am|pm)?/i);
      if (timeParts) {
        hours = parseInt(timeParts[1]);
        minutes = parseInt(timeParts[2]);
        
        if (timeParts[3] && timeParts[3].toLowerCase() === 'pm' && hours < 12) {
          hours += 12;
        } else if (timeParts[3] && timeParts[3].toLowerCase() === 'am' && hours === 12) {
          hours = 0;
        }
      } else {
        const simpleTime = time.match(/(\d{1,2})\s*(am|pm)/i);
        if (simpleTime) {
          hours = parseInt(simpleTime[1]);
          
          if (simpleTime[2].toLowerCase() === 'pm' && hours < 12) {
            hours += 12;
          } else if (simpleTime[2].toLowerCase() === 'am' && hours === 12) {
            hours = 0;
          }
        }
      }
      
      dueDate.setHours(hours, minutes, 0, 0);
    } else if (dateStr.toLowerCase() === 'today') {
      // For "today" tasks WITHOUT a specified time, keep the current time
      // (we don't modify the time)
    } else {
      // For non-today tasks without specific time, set to 9am
      dueDate.setHours(9, 0, 0, 0);
    }
    
    console.log(`Parsed task due date: ${dueDate.toISOString()} from "${dateStr}" and "${timeStr}"`);
    return dueDate;
  } catch (error) {
    console.error(`Error parsing due date: ${error}`);
    return new Date(); // Default to today
  }
}

// Send an email notification for new events and tasks
function sendEmailNotification(events, log) {
  if (!events || events.length === 0) return;
  
  // Get the user's email (the person who owns the script)
  const email = Session.getActiveUser().getEmail();
  
  // Create the email subject
  const subject = `New Life Logs Events/Tasks Found: ${events.length} items`;
  
  // Create the email body with HTML formatting
  let body = `<h2>New Events/Tasks from Life Logs</h2>`;
  body += `<p>The following ${events.length} items were extracted from your recent Life Logs:</p>`;
  
  // Add a table of events
  body += `<table border="1" cellpadding="5" style="border-collapse: collapse;">`;
  body += `<tr style="background-color: #f2f2f2;"><th>Type</th><th>Title</th><th>Date</th><th>Time</th><th>Details</th></tr>`;
  
  // Add each event to the table
  for (const event of events) {
    const type = event.isReminder ? "Task" : "Calendar Event";
    const bgColor = event.isReminder ? "#fff0f0" : "#f0f0ff"; // Light red for tasks, light blue for events
    
    body += `<tr style="background-color: ${bgColor};">`;
    body += `<td><strong>${type}</strong></td>`;
    body += `<td>${event.title}</td>`;
    body += `<td>${event.date}</td>`;
    body += `<td>${event.time || "No time specified"}</td>`;
    body += `<td>${event.details || ""}</td>`;
    body += `</tr>`;
  }
  
  body += `</table>`;
  
  // Add information about the source log
  body += `<p><strong>Source:</strong> "${log.title}" (${new Date(log.startTime).toLocaleString()})</p>`;
  
  // Add a footer with links to view tasks/calendar
  body += `<p style="margin-top: 20px;">`;
  body += `<a href="https://calendar.google.com/" style="margin-right: 15px;">View Calendar</a> | `;
  body += `<a href="https://tasks.google.com/">View Tasks</a>`;
  body += `</p>`;
  
  // Send the email
  try {
    MailApp.sendEmail({
      to: email,
      subject: subject,
      htmlBody: body
    });
    console.log(`Email notification sent to ${email}`);
  } catch (error) {
    console.error("Error sending email notification:", error);
  }
}

// Function to test the script with a sample log
function testWithSampleLog() {
  const sampleLog = {
    id: "test-log",
    title: "Test Log",
    startTime: new Date().toISOString(),
    markdown: `
      Speaker 1: We have you booked in for the 14th of November at 3:30 PM for your dental checkup.
      
      Speaker 2: Great, thanks for confirming. I'll make sure to be there.
      
      Speaker 1: Don't forget to get milk today, we're completely out and I need it for breakfast tomorrow.
      
      Speaker 2: Okay, I'll pick some up from the store on my way home.
      
      Speaker 1: Also, remember we have dinner with the Johnsons next Friday at 7:00 PM.
    `
  };
  
  console.log("Testing with sample log...");
  
  // Make sure we have access to Tasks API
  try {
    // Enable the Tasks API
    enableTasksService();
    
    // First, verify the OpenAI connection
    console.log("First, testing API connection...");
    if (!testOpenAIConnection()) {
      console.log("API connection test failed. Please check your OpenAI API key.");
      return;
    }
    
    console.log("Now extracting events from log...");
    const events = extractEventsWithOpenAI(sampleLog.markdown);
    
    if (events && events.length > 0) {
      console.log(`Found ${events.length} events in sample log:`, events);
      console.log("Testing adding events to calendar and tasks...");
      processEvents(events, sampleLog);
      console.log("Test completed successfully!");
    } else {
      console.log("No events found in sample log or API error occurred.");
    }
  } catch (e) {
    console.error("Error enabling Tasks service:", e);
    console.log("Please enable the Google Tasks API manually in Resources > Advanced Google services...");
  }
}

// Enable Tasks API service
function enableTasksService() {
  // Check if Tasks API is already enabled
  try {
    Tasks.Tasklists.list();
    console.log("Tasks API is already enabled.");
  } catch (e) {
    console.log("Tasks API is not enabled. You need to enable it manually.");
    console.log("Go to Resources > Advanced Google services... and enable the Tasks API.");
    throw new Error("Tasks API not enabled");
  }
}

// Simple function to test just the OpenAI API connection
function testOpenAIConnection() {
  console.log("Testing OpenAI API connection...");
  
  const options = {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${OPENAI_API_KEY}`
    },
    payload: JSON.stringify({
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "user",
          content: "Say 'API connection successful'"
        }
      ],
      max_tokens: 50
    }),
    muteHttpExceptions: true
  };
  
  try {
    const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
    const responseCode = response.getResponseCode();
    console.log(`OpenAI API response code: ${responseCode}`);
    
    const responseText = response.getContentText();
    console.log("Response:", responseText);
    
    if (responseCode === 200) {
      console.log("✅ OpenAI API connection is working!");
      return true;
    } else {
      console.log("❌ OpenAI API connection failed");
      return false;
    }
  } catch (error) {
    console.error("❌ Error testing OpenAI API connection:", error);
    return false;
  }
}
```

### Step 4: Add Your API Keys

Replace the placeholders at the top of the script with your actual API keys:

```javascript
const LIMITLESS_API_KEY = "your-limitless-api-key-here";
const OPENAI_API_KEY = "your-openai-api-key-here";
```

### Step 5: Enable Required Google Services

1. In the left sidebar of the Apps Script editor, click on "+" next to "Services"
2. Find and add these services:
   - **Gmail** (for sending email notifications)
   - **Calendar** (for adding calendar events)
   - **Tasks** (for adding tasks)
3. Click "Add" for each service

### Step 6: Test the Integration

1. Click the dropdown menu next to "Debug" and select `testWithSampleLog`
2. Click the "Run" button
3. When prompted, click "Review permissions" and allow the script to access your Google account
4. Check the logs to see if the test is successful
5. Verify that:
   - A new "Life Logs Appointments" calendar was created (if it didn't exist before)
   - Sample events were added to your calendar
   - Sample tasks were added to your Google Tasks
   - You received an email notification

### Step 7: Set Up Automatic Execution

1. Click on the clock icon in the left sidebar (Triggers)
2. Click "Add Trigger" at the bottom right
3. Configure the trigger:
   - Choose function: `processLifeLogs`
   - Choose deployment: "Head"
   - Event source: "Time-driven"
   - Type of time: "Minutes timer"
   - Minutes interval: "Every 5 minutes" (or your preferred frequency)
4. Click "Save"

## How It Works

### Calendar Events vs. Tasks

The script categorizes items it finds into two types:

1. **Calendar Events** (when `isReminder` is `false`):
   - Added to Google Calendar
   - Default to 7:00 AM if no time is specified
   - Include a 30-minute reminder notification

2. **Tasks/Reminders** (when `isReminder` is `true`):
   - Added to Google Tasks
   - Marked with high priority (⚠️)
   - Set to current time for "today" tasks

### Special Cases

- **Appointments with no date**: Converted to high-priority tasks titled "Follow up about: [appointment name]"
- **Today's tasks**: Use the current time to ensure they appear immediately
- **Completed tasks**: Not re-added if you've already marked them as complete

## Customization

You can customize the script by modifying these settings near the top:

```javascript
// How many hours back to check for new logs
const HOURS_LOOKBACK = 24;

// Calendar settings
const CALENDAR_NAME = "Life Logs Appointments"; 
const DEFAULT_REMINDER_MINUTES = 30;
```

## Troubleshooting

### Common Issues

1. **API Key Issues**:
   - Run `testOpenAIConnection()` to check your OpenAI API key
   - Check the Limitless API documentation for key format

2. **Missing Calendar or Tasks**:
   - Ensure you've granted the necessary permissions
   - Check if you have multiple Google accounts and are using the right one

3. **No Events Being Found**:
   - Check the logs to see how the OpenAI API is responding
   - Try modifying the prompt in the `extractEventsWithOpenAI` function

4. **Script Timing Out**:
   - Google Apps Script has a 6-minute execution limit
   - Reduce the number of logs processed in one go (reduce `HOURS_LOOKBACK`)

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Credits

- Created by [Your Name]
- Uses OpenAI's GPT-3.5 Turbo model for event extraction
- Integrates with Limitless Life Logs, Google Calendar, and Google Tasks
