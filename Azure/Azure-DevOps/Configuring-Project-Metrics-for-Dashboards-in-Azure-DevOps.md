# Configuring Project Metrics for Dashboards in Azure DevOps: Measuring Work Item Lead Time and Cycle Time Using REST API

## Introduction
In agile development environments, understanding the time metrics of work items is crucial for identifying process bottlenecks and driving continuous improvement. While Azure DevOps provides a UI-based approach to configure dashboards *(Reference [link](https://learn.microsoft.com/en-us/azure/devops/report/dashboards/cycle-time-and-lead-time?view=azure-devops))*, using the REST API offers greater flexibility and automation capabilities. This blog will focus on how to programmatically configure chart widgets that measure both lead time and cycle time for work items.

## Understanding Lead Time vs. Cycle Time
Both metrics are critical for assessing team efficiency but measure different aspects of your workflow:
- **Lead Time:** The total time from when a work item is created until it's completed, representing the end-to-end customer experience.
- **Cycle Time:** The period from when a work item becomes active (work begins) until it's completed, representing the team's execution efficiency.

By monitoring both metrics, you can:
- Identify bottlenecks in your development process
- Measure efficiency at different stages of your workflow
- Predict completion times for future work items
- Set realistic expectations with stakeholders
  
## Using REST API to Add Lead Time and Cycle Time Widgets
### 1. Prerequisites
Before getting started, ensure you have:

- Personal Access Token (PAT) with appropriate permissions
- Organization name and project ID
- Team ID (if configuring for a specific team)
- Dashboard ID where you want to add the widgets

### 2. Authentication Setup

First, set up your authentication headers:

```javascript
const headers = {
  'Content-Type': 'application/json',
  'Authorization': 'Basic ' + Buffer.from(':' + personalAccessToken).toString('base64')
};
```

### 3. API Endpoint for Widget Creation

The REST API endpoint for adding a widget to a dashboard is:

```bash
POST https://dev.azure.com/{organization}/{project}/{team}/_apis/dashboard/dashboards/{dashboardId}/widgets?api-version=7.1-preview.2
```

### 4. Create the Lead Time Widget with REST API

Here's a sample JSON payload to create a Lead Time widget:

```js
const leadTimeWidgetPayload = {
  name: "Lead Time Widget",
  position: {
    row: 0,
    column: 0
  },
  size: {
    rowSpan: 2,
    columnSpan: 4
  },
  settings: JSON.stringify({
    title: "Work Item Lead Time",
    teamId: "your-team-id",
    workItemType: "User Story", // Or other work item types
    aggregation: "avg", // avg, median, or percentile
    timeFrame: "30", // Last 30 days
    showLeadTime: true,
    showCycleTime: false,
    advancedSettings: {
      filters: {
        tags: ["important", "critical"], // Optional
        areaPaths: ["Area\\Path1"], // Optional
        iterationPaths: ["Iteration\\Path1"] // Optional
      },
      thresholds: [
        {
          value: 10, // 10 days threshold
          color: "#81C784" // Green
        },
        {
          value: 20, // 20 days threshold
          color: "#FFB74D" // Orange
        }
      ]
    }
  }),
  contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.LeadTimeWidget"
};
```

### 5. Create the Cycle Time Widget with REST API

Here's a sample JSON payload to create a Cycle Time widget:
```js
const cycleTimeWidgetPayload = {
  name: "Cycle Time Widget",
  position: {
    row: 0,
    column: 4 // Position next to Lead Time widget
  },
  size: {
    rowSpan: 2,
    columnSpan: 4
  },
  settings: JSON.stringify({
    title: "Work Item Cycle Time",
    teamId: "your-team-id",
    workItemType: "User Story", // Or other work item types
    aggregation: "avg", // avg, median, or percentile
    timeFrame: "30", // Last 30 days
    showLeadTime: false,
    showCycleTime: true,
    advancedSettings: {
      filters: {
        tags: ["important", "critical"], // Optional
        areaPaths: ["Area\\Path1"], // Optional
        iterationPaths: ["Iteration\\Path1"] // Optional
      },
      thresholds: [
        {
          value: 5, // 5 days threshold
          color: "#81C784" // Green
        },
        {
          value: 10, // 10 days threshold
          color: "#FFB74D" // Orange
        }
      ]
    }
  }),
  contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.CycleTimeWidget"
};
```

### 6. Create a Combined Lead Time and Cycle Time Widget

Alternatively, you can create a single widget that displays both metrics:

```js
const combinedWidgetPayload = {
  name: "Lead Time and Cycle Time",
  position: {
    row: 0,
    column: 0
  },
  size: {
    rowSpan: 2,
    columnSpan: 8
  },
  settings: JSON.stringify({
    title: "Work Item Time Metrics",
    teamId: "your-team-id",
    workItemType: "User Story", // Or other work item types
    aggregation: "avg", // avg, median, or percentile
    timeFrame: "30", // Last 30 days
    showLeadTime: true,
    showCycleTime: true,
    advancedSettings: {
      filters: {
        tags: ["important", "critical"], // Optional
        areaPaths: ["Area\\Path1"], // Optional
        iterationPaths: ["Iteration\\Path1"] // Optional
      },
      thresholds: [
        {
          value: 5, // 5 days threshold for cycle time
          value2: 10, // 10 days threshold for lead time
          color: "#81C784" // Green
        },
        {
          value: 10, // 10 days threshold for cycle time
          value2: 20, // 20 days threshold for lead time
          color: "#FFB74D" // Orange
        }
      ]
    }
  }),
  contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.LeadCycleTimeWidget"
};
```

### 7. Make the API Request

Using your preferred programming language, make the REST API call to create the widgets:
```js
const axios = require('axios');

async function addTimeMetricWidgets() {
  try {
    // Add Lead Time widget
    const leadTimeResponse = await axios.post(
      `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards/${dashboardId}/widgets?api-version=7.1-preview.2`,
      leadTimeWidgetPayload,
      { headers }
    );
    
    console.log('Lead Time widget created successfully:', leadTimeResponse.data);
    
    // Add Cycle Time widget
    const cycleTimeResponse = await axios.post(
      `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards/${dashboardId}/widgets?api-version=7.1-preview.2`,
      cycleTimeWidgetPayload,
      { headers }
    );
    
    console.log('Cycle Time widget created successfully:', cycleTimeResponse.data);
    
    return {
      leadTimeWidget: leadTimeResponse.data,
      cycleTimeWidget: cycleTimeResponse.data
    };
  } catch (error) {
    console.error('Error creating widgets:', error.response ? error.response.data : error.message);
    throw error;
  }
}
```

## Sharing Dashboard Templates Across Teams
A key advantage of using REST API for dashboard configuration is the ability to easily share these configurations across teams. Here are four methods for sharing your dashboard templates:

### Method 1: Process Template Method
Azure DevOps allows you to include dashboard configurations in process templates, which is the official way to share standardized configurations:

**1. Export Existing Dashboard Configuration**
First, retrieve the configuration of your existing dashboard:
```js
async function exportDashboardConfig(organization, project, team, dashboardId) {
  try {
    const response = await axios.get(
      `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards/${dashboardId}?api-version=7.1-preview.2`,
      { headers }
    );
    
    // Get all widgets on the dashboard
    const widgetsResponse = await axios.get(
      `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards/${dashboardId}/widgets?api-version=7.1-preview.2`,
      { headers }
    );
    
    // Combine dashboard and widget configurations
    const completeConfig = {
      dashboard: response.data,
      widgets: widgetsResponse.data.value
    };
    
    // Save configuration to the file system
    fs.writeFileSync('dashboard-template.json', JSON.stringify(completeConfig, null, 2));
    
    return completeConfig;
  } catch (error) {
    console.error('Error exporting dashboard configuration:', error);
    throw error;
  }
}
```
**2. Create Process Template XML**
Convert the exported dashboard configuration to process template XML format:
```js
function convertToProcessTemplate(dashboardConfig) {
  // Process template XML structure
  const processTemplateXml = `<?xml version="1.0" encoding="utf-8"?>
<process name="LeadCycleTimeTemplate" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <dashboards>
    <dashboard name="${dashboardConfig.dashboard.name}">
      <description>${dashboardConfig.dashboard.description || ''}</description>
      ${dashboardConfig.widgets.map(widget => `
      <widget name="${widget.name}" position-x="${widget.position.column}" position-y="${widget.position.row}" size-x="${widget.size.columnSpan}" size-y="${widget.size.rowSpan}">
        <contributionId>${widget.contributionId}</contributionId>
        <settings>
          ${JSON.stringify(JSON.parse(widget.settings))}
        </settings>
      </widget>`).join('')}
    </dashboard>
  </dashboards>
</process>`;

  fs.writeFileSync('LeadCycleTimeTemplate.xml', processTemplateXml);
  return processTemplateXml;
}
```

**3. Upload the Template to Organization-Level Process**
Upload the template to the organization-level process library through the Azure DevOps administration interface or using REST API.

### Method 2: Dashboard-as-Code Method
Provide teams with a runnable script that creates standardized dashboards with a single command:

**1. Create a Reusable Dashboard Creation Script**
```js
// dashboardCreator.js
const axios = require('axios');

// Standardized configurations
const standardWidgets = [
  {
    name: "Lead Time Widget",
    contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.LeadTimeWidget",
    position: { row: 0, column: 0 },
    size: { rowSpan: 2, columnSpan: 4 },
    settings: {
      title: "Work Item Lead Time",
      workItemType: "User Story",
      // Other standardized settings...
    }
  },
  {
    name: "Cycle Time Widget",
    contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.CycleTimeWidget",
    position: { row: 0, column: 4 },
    size: { rowSpan: 2, columnSpan: 4 },
    settings: {
      title: "Work Item Cycle Time",
      workItemType: "User Story",
      // Other standardized settings...
    }
  }
  // Add more standardized widgets...
];

async function createStandardDashboard(options) {
  const { pat, organization, project, team, dashboardName } = options;
  
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Basic ' + Buffer.from(':' + pat).toString('base64')
  };
  
  try {
    // Create new dashboard
    const dashboardPayload = {
      name: dashboardName || "Flow Metrics Dashboard",
      description: "Standardized metrics dashboard with Lead Time and Cycle Time"
    };
    
    const dashboardResponse = await axios.post(
      `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards?api-version=7.1-preview.2`,
      dashboardPayload,
      { headers }
    );
    
    const dashboardId = dashboardResponse.data.id;
    
    // Add standardized widgets to the dashboard
    for (const widgetTemplate of standardWidgets) {
      // Customize settings for the specific team
      const widgetSettings = {
        ...widgetTemplate.settings,
        teamId: team // Ensure it's for the correct team
      };
      
      const widgetPayload = {
        name: widgetTemplate.name,
        position: widgetTemplate.position,
        size: widgetTemplate.size,
        settings: JSON.stringify(widgetSettings),
        contributionId: widgetTemplate.contributionId
      };
      
      await axios.post(
        `https://dev.azure.com/${organization}/${project}/${team}/_apis/dashboard/dashboards/${dashboardId}/widgets?api-version=7.1-preview.2`,
        widgetPayload,
        { headers }
      );
    }
    
    console.log(`Successfully created standardized dashboard for ${team}`);
    return dashboardId;
  } catch (error) {
    console.error('Error creating standardized dashboard:', error.response ? error.response.data : error.message);
    throw error;
  }
}

module.exports = { createStandardDashboard };
```

**2. Create a Simple Interface for Teams to Use**

```js
// createDashboard.js
const { createStandardDashboard } = require('./dashboardCreator');
const readline = require('readline');

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

async function promptAndCreate() {
  const pat = await question('Enter your Personal Access Token (PAT): ');
  const organization = await question('Enter organization name: ');
  const project = await question('Enter project name: ');
  const team = await question('Enter team name: ');
  const dashboardName = await question('Enter dashboard name (default is "Flow Metrics Dashboard"): ');
  
  try {
    const dashboardId = await createStandardDashboard({
      pat,
      organization,
      project,
      team,
      dashboardName: dashboardName || undefined
    });
    
    console.log(`Dashboard created successfully! Access it at: https://dev.azure.com/${organization}/${project}/_dashboards/dashboard/${dashboardId}`);
  } catch (error) {
    console.error('Failed to create dashboard:', error);
  } finally {
    rl.close();
  }
}

function question(query) {
  return new Promise(resolve => {
    rl.question(query, resolve);
  });
}

promptAndCreate();
```

### Method 3: Azure DevOps Extension Method
Create a custom Azure DevOps extension for your organization that provides a one-click dashboard creation experience:

**1. Create Extension Infrastructure**
Create a basic extension structure, including a `vss-extension.json` file:
```json
{
  "manifestVersion": 1,
  "id": "standard-dashboard-creator",
  "name": "Standard Dashboard Creator",
  "version": "1.0.0",
  "publisher": "YourPublisherName",
  "description": "Create standardized Lead Time and Cycle Time metrics dashboards",
  "targets": [
    {
      "id": "Microsoft.VisualStudio.Services"
    }
  ],
  "contributions": [
    {
      "id": "create-standard-dashboard-hub",
      "type": "ms.vss-web.hub",
      "targets": [
        "ms.vss-web.project-admin-hub-group"
      ],
      "properties": {
        "name": "Standard Dashboards",
        "uri": "dashboard-creator.html"
      }
    }
  ],
  "files": [
    {
      "path": "dashboard-creator.html",
      "addressable": true
    },
    {
      "path": "scripts",
      "addressable": true
    },
    {
      "path": "images",
      "addressable": true
    }
  ]
}
```

**2. Create the User Interface**
Create a user-friendly interface in `dashboard-creator.html`:
```html
<!DOCTYPE html>
<html>
<head>
  <script src="sdk/VSS.SDK.min.js"></script>
  <script>
    VSS.init({
      explicitNotifyLoaded: true,
      usePlatformScripts: true
    });
    
    VSS.require(["scripts/dashboard-creator"], function(creator) {
      creator.initialize();
      VSS.notifyLoadSucceeded();
    });
  </script>
</head>
<body>
  <div class="dashboard-creator-container">
    <h2>Create Standard Metrics Dashboard</h2>
    <p>Create a standardized dashboard with Lead Time and Cycle Time widgets for the current team.</p>
    
    <div class="form-group">
      <label for="dashboard-name">Dashboard Name:</label>
      <input type="text" id="dashboard-name" value="Flow Metrics Dashboard" class="form-control" />
    </div>
    
    <button id="create-dashboard" class="btn btn-primary">Create Dashboard</button>
    
    <div id="status-message" style="margin-top: 20px;"></div>
  </div>
</body>
</html>
```

**3.Implement Extension Logic**
Create a `scripts/dashboard-creator.js` file:
```js
define(["VSS/Service", "TFS/Core/RestClient", "TFS/Dashboards/RestClient"], 
  function(VSS_Service, TFS_Core_RestClient, TFS_Dashboards_RestClient) {
  
  var dashboardCreator = {
    initialize: function() {
      this.attachEvents();
    },
    
    attachEvents: function() {
      $("#create-dashboard").click(this.createDashboard.bind(this));
    },
    
    createDashboard: async function() {
      try {
        $("#status-message").text("Creating dashboard...");
        
        const dashboardName = $("#dashboard-name").val();
        
        // Get current context
        const context = VSS.getWebContext();
        const projectId = context.project.id;
        const teamId = context.team.id;
        
        // Get REST clients
        const dashboardClient = TFS_Dashboards_RestClient.getClient();
        
        // Create dashboard
        const dashboard = await dashboardClient.createDashboard({
          name: dashboardName,
          description: "Standardized metrics dashboard with Lead Time and Cycle Time",
          widgets: []
        }, projectId, teamId);
        
        // Add Lead Time widget
        await dashboardClient.createWidget({
          name: "Lead Time Widget",
          position: { row: 0, column: 0 },
          size: { rowSpan: 2, columnSpan: 4 },
          settings: JSON.stringify({
            title: "Work Item Lead Time",
            teamId: teamId,
            workItemType: "User Story",
            aggregation: "avg",
            timeFrame: "30",
            showLeadTime: true,
            showCycleTime: false
          }),
          contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.LeadTimeWidget"
        }, projectId, teamId, dashboard.id);
        
        // Add Cycle Time widget
        await dashboardClient.createWidget({
          name: "Cycle Time Widget",
          position: { row: 0, column: 4 },
          size: { rowSpan: 2, columnSpan: 4 },
          settings: JSON.stringify({
            title: "Work Item Cycle Time",
            teamId: teamId,
            workItemType: "User Story",
            aggregation: "avg",
            timeFrame: "30",
            showLeadTime: false,
            showCycleTime: true
          }),
          contributionId: "ms.vss-dashboards-web.Microsoft.VisualStudioOnline.Dashboards.CycleTimeWidget"
        }, projectId, teamId, dashboard.id);
        
        $("#status-message").html(`<div class="success">Dashboard created successfully! <a href="#" onclick="window.open('${context.host.uri}${context.project.name}/_dashboards/dashboard/${dashboard.id}', '_blank')">Click to view</a></div>`);
      } catch (error) {
        $("#status-message").html(`<div class="error">Failed to create dashboard: ${error.message}</div>`);
        console.error(error);
      }
    }
  };
  
  return dashboardCreator;
});
```

### Method 4: Fully Automated Deployment with Event Triggers
Create an automated system that deploys standardized dashboards whenever a new team is created:


**1. Create Azure Function for Automated Deployment**
```js
// deploy-dashboards.js
const axios = require('axios');
const fs = require('fs');
const path = require('path');

// Dashboard deployer class
class DashboardDeployer {
  constructor(config) {
    this.organization = config.organization;
    this.project = config.project;
    this.pat = config.pat;
    this.teamId = config.teamId;
    this.teamName = config.teamName;
    
    // Setup auth headers
    this.headers = {
      'Content-Type': 'application/json',
      'Authorization': 'Basic ' + Buffer.from(':' + this.pat).toString('base64')
    };
    
    // Load widget templates
    const templatePath = path.resolve(__dirname, './widget-templates.json');
    this.widgetTemplates = JSON.parse(fs.readFileSync(templatePath, 'utf8'));
    
    // Dashboard name
    this.dashboardName = config.dashboardName || "Standard Metrics Dashboard";
    this.dashboardDescription = config.dashboardDescription || "Standardized dashboard with Lead Time and Cycle Time metrics";
  }
  
  // Check if team already has a dashboard with the same name
  async checkExistingDashboard() {
    try {
      const response = await axios.get(
        `https://dev.azure.com/${this.organization}/${this.project}/${this.teamName}/_apis/dashboard/dashboards?api-version=7.1-preview.2`,
        { headers: this.headers }
      );
      
      const existingDashboard = response.data.value.find(d => d.name === this.dashboardName);
      return existingDashboard;
    } catch (error) {
      console.error(`Failed to get dashboards for team ${this.teamName}:`, error.response?.data || error.message);
      return null;
    }
  }
  
  // Create dashboard
  async createDashboard() {
    try {
      console.log(`Creating dashboard for team ${this.teamName}...`);
      
      // Check if dashboard already exists
      const existingDashboard = await this.checkExistingDashboard();
      if (existingDashboard) {
        console.log(`Team ${this.teamName} already has a dashboard named "${this.dashboardName}", skipping creation...`);
        return { id: existingDashboard.id, isNew: false };
      }
      
      // Create new dashboard
      const dashboardResponse = await axios.post(
        `https://dev.azure.com/${this.organization}/${this.project}/${this.teamName}/_apis/dashboard/dashboards?api-version=7.1-preview.2`,
        {
          name: this.dashboardName,
          description: this.dashboardDescription
        },
        { headers: this.headers }
      );
      
      console.log(`Successfully created dashboard for team ${this.teamName}, ID: ${dashboardResponse.data.id}`);
      return { id: dashboardResponse.data.id, isNew: true };
    } catch (error) {
      console.error(`Failed to create dashboard for team ${this.teamName}:`, error.response?.data || error.message);
      throw error;
    }
  }
  
  // Add widgets to dashboard
  async addWidgetsToDashboard(dashboardId) {
    try {
      console.log(`Adding widgets to dashboard for team ${this.teamName}...`);
      
      for (const [index, template] of this.widgetTemplates.entries()) {
        console.log(`Adding widget ${index + 1}/${this.widgetTemplates.length}: ${template.name}`);
        
        // Copy template and update settings
        const widgetPayload = { ...template };
        
        // Parse settings and update teamId
        const settings = JSON.parse(widgetPayload.settings);
        settings.teamId = this.teamId;
        widgetPayload.settings = JSON.stringify(settings);
        
        // Add widget
        await axios.post(
          `https://dev.azure.com/${this.organization}/${this.project}/${this.teamName}/_apis/dashboard/dashboards/${dashboardId}/widgets?api-version=7.1-preview.2`,
          widgetPayload,
          { headers: this.headers }
        );
      }
      
      console.log(`Successfully added widgets to dashboard for team ${this.teamName}`);
    } catch (error) {
      console.error(`Failed to add widgets for team ${this.teamName}:`, error.response?.data || error.message);
      throw error;
    }
  }
  
  // Deploy dashboard
  async deploy() {
    try {
      // Create dashboard
      const { id: dashboardId, isNew } = await this.createDashboard();
      
      // If it's a new dashboard, add widgets
      if (isNew) {
        await this.addWidgetsToDashboard(dashboardId);
        
        return {
          team: this.teamName,
          dashboardId,
          success: true,
          isNew: true,
          url: `https://dev.azure.com/${this.organization}/${this.project}/_dashboards/dashboard/${dashboardId}`
        };
      } else {
        return {
          team: this.teamName,
          dashboardId,
          success: true,
          isNew: false
        };
      }
    } catch (error) {
      console.error(`Dashboard deployment failed for team ${this.teamName}:`, error);
      return {
        team: this.teamName,
        success: false,
        error: error.message
      };
    }
  }
}

// Azure Function main handler
module.exports = async function (context, req) {
  context.log('Azure DevOps Team Creation Trigger activated');
  
  try {
    // Validate request
    if (!req.body || !req.body.eventType) {
      context.res = {
        status: 400,
        body: "Invalid payload - missing eventType"
      };
      return;
    }
    
    // Confirm this is a team creation event
    const eventType = req.body.eventType;
    if (eventType !== 'team.created') {
      context.log(`Received non-team creation event: ${eventType}, ignoring`);
      context.res = {
        status: 200,
        body: `Received event ${eventType}, not a team creation event. Ignoring.`
      };
      return;
    }
    
    // Extract necessary information
    const teamInfo = req.body.resource;
    const projectInfo = req.body.resourceContainers.project;
    
    context.log(`Detected new team creation: ${teamInfo.name} (ID: ${teamInfo.id}) in project ${projectInfo.name}`);
    
    // Get PAT from environment variable
    const pat = process.env.AZURE_DEVOPS_PAT;
    if (!pat) {
      throw new Error("Missing AZURE_DEVOPS_PAT environment variable");
    }
    
    // Parse organization name from request URL
    const organizationUrl = req.body.organizationUrl || '';
    const organization = organizationUrl.split('/').pop();
    
    if (!organization) {
      throw new Error("Could not determine organization from request");
    }
    
    // Create deployer configuration
    const config = {
      organization,
      project: projectInfo.name,
      pat,
      teamId: teamInfo.id,
      teamName: teamInfo.name
    };
    
    // Deploy dashboard
    const deployer = new DashboardDeployer(config);
    const result = await deployer.deploy();
    
    if (result.success) {
      if (result.isNew) {
        context.log(`Successfully created dashboard for new team ${teamInfo.name}: ${result.dashboardId}`);
        context.res = {
          status: 200,
          body: {
            message: `Successfully deployed dashboard for team ${teamInfo.name}`,
            dashboardId: result.dashboardId,
            dashboardUrl: result.url
          }
        };
      } else {
        context.log(`Team ${teamInfo.name} already has a dashboard, deployment skipped`);
        context.res = {
          status: 200,
          body: {
            message: `Team ${teamInfo.name} already has a dashboard, deployment skipped`,
            dashboardId: result.dashboardId
          }
        };
      }
    } else {
      throw new Error(`Dashboard deployment failed: ${result.error}`);
    }
  } catch (error) {
    context.log.error(`Error occurred: ${error.message}`);
    context.res = {
      status: 500,
      body: {
        error: `Failed to deploy dashboard: ${error.message}`
      }
    };
  }
};
```
**2. Create Azure DevOps Webhook**
Configure a webhook in Azure DevOps organization settings:
1. Go to Organization Settings > Service Hooks
2. Create a new webhook subscription
3. Select "Team created" as the trigger event
4. Point it to your Azure Function URL
5. Test and save the configuration

**3. Setup Instructions for Teams**
Create clear instructions for how to configure the automated system:

1. Deploy the Azure Function with the widget templates
2. Configure the webhook in Azure DevOps
3. Create a new team to test the automation
4. Verify the dashboard is automatically created

## Comparing the Four Methods

Each method has its own strengths and is suitable for different organizational needs

### Method 1: Process Template Method
**Best for:** Organizations with standardized project templates and formal processes
- **Pros:** Official Azure DevOps approach, integrated with project creation flows
- **Cons:** Less flexible for updates, requires admin privileges to modify templates

### Method 2: Dashboard-as-Code Method
**Best for:** DevOps-oriented teams comfortable with scripts

- **Pros:** Easy to version control, highly customizable, simple for teams to use
- **Cons:** Requires running scripts, PAT management for each user

### Method 3: Azure DevOps Extension Method
**Best for:** Large organizations with many teams and non-technical users

- **Pros:** User-friendly interface, no coding required for end users, centralized deployment
- **Cons:** Requires extension development and publishing

### Method 4: Fully Automated Deployment with Event Triggers
**Best for:** Enterprise environments with high standardization requirements and frequent team creation

- **Pros:** Zero user intervention, consistent implementation, scales automatically with organization growth
- **Cons:** More complex initial setup, requires Azure Functions and webhook configuration

### Best Practices for Dashboard Standardization
Regardless of the method you choose, follow these best practices:

1. **Document the Metrics:** Clearly define what Lead Time and Cycle Time mean in your organization
2. **Standardize Thresholds:** Establish consistent color-coding and threshold values across all teams
3. **Review Regularly:** Schedule periodic reviews of the dashboard templates to ensure they remain relevant
4. **Gather Feedback:** Collect input from teams about the usefulness and accuracy of the metrics
5. **Provide Training:** Ensure all teams understand how to interpret and act
6. **Maintain Data Quality:** Ensure work item state transitions are accurately recorded to get reliable Lead Time and Cycle Time data
7. **Focus on Trends:** Emphasize trend analysis over absolute values, as different teams and work types will have different baseline metrics
8. **Link to Actions:** Connect the metrics to specific improvement actions in retrospectives and planning meetings
9. **Consider Team Context:** Allow for some customization based on team-specific factors while maintaining core metric consistency
10. **Centralize Updates:** Create a mechanism to update all dashboards when the organization decides to modify the template

## Retrieving and Analyzing Time Metric Data
Once your dashboards are configured, you can retrieve the raw time metric data for advanced analysis:

### Using Analytics Query API
```js
const oDataQuery = {
  query: `
    WorkItems
    ?$filter=WorkItemType eq 'User Story' and State eq 'Completed' and CompletedDate ge ${startDate} and CompletedDate le ${endDate}
    &$select=WorkItemId,Title,State,StateCategory,CreatedDate,ActivatedDate,CompletedDate
    &$orderby=CompletedDate desc
  `
};

// Make the API request to get work item data
const analyticsResponse = await axios.post(
  `https://analytics.dev.azure.com/${organization}/${project}/_odata/v4.0-preview/WorkItems`,
  oDataQuery,
  { headers }
);

// Calculate both Lead Time and Cycle Time
function calculateTimeMetrics(workItems) {
  return workItems.map(item => {
    const createdDate = new Date(item.CreatedDate);
    const activatedDate = new Date(item.ActivatedDate);
    const completedDate = new Date(item.CompletedDate);
    
    const leadTimeInDays = (completedDate - createdDate) / (1000 * 60 * 60 * 24);
    const cycleTimeInDays = (completedDate - activatedDate) / (1000 * 60 * 60 * 24);
    
    return {
      id: item.WorkItemId,
      title: item.Title,
      leadTimeInDays: Math.round(leadTimeInDays * 10) / 10, // Round to 1 decimal place
      cycleTimeInDays: Math.round(cycleTimeInDays * 10) / 10, // Round to 1 decimal place
      createdDate,
      activatedDate,
      completedDate
    };
  });
}
```

### Advanced Metrics Derived from Lead Time and Cycle Time
With both metrics, you can calculate more advanced metrics to gain deeper insights:

### Queue Time (Wait Time)
Calculate how long work items wait before active development begins:
```js
function calculateQueueTime(workItems) {
  return workItems.map(item => {
    const createdDate = new Date(item.CreatedDate);
    const activatedDate = new Date(item.ActivatedDate);
    
    const queueTimeInDays = (activatedDate - createdDate) / (1000 * 60 * 60 * 24);
    
    return {
      id: item.WorkItemId,
      title: item.Title,
      queueTimeInDays: Math.round(queueTimeInDays * 10) / 10
    };
  });
}
```

### Process Efficiency Ratio
Calculate the ratio of active work time to total lead time:

```js
function calculateEfficiencyRatio(workItems) {
  return workItems.map(item => {
    const createdDate = new Date(item.CreatedDate);
    const activatedDate = new Date(item.ActivatedDate);
    const completedDate = new Date(item.CompletedDate);
    
    const leadTimeInDays = (completedDate - createdDate) / (1000 * 60 * 60 * 24);
    const cycleTimeInDays = (completedDate - activatedDate) / (1000 * 60 * 60 * 24);
    
    // Efficiency ratio: proportion of time spent actively working on item
    const efficiencyRatio = cycleTimeInDays / leadTimeInDays;
    
    return {
      id: item.WorkItemId,
      title: item.Title,
      efficiencyRatio: Math.round(efficiencyRatio * 100) // Percentage
    };
  });
}
```

### Predictability Metrics
Analyze the variability of your time metrics to understand predictability:
```js
function calculatePredictabilityMetrics(workItems) {
  // Calculate lead times and cycle times
  const leadTimes = workItems.map(item => {
    const createdDate = new Date(item.CreatedDate);
    const completedDate = new Date(item.CompletedDate);
    return (completedDate - createdDate) / (1000 * 60 * 60 * 24);
  });
  
  const cycleTimes = workItems.map(item => {
    const activatedDate = new Date(item.ActivatedDate);
    const completedDate = new Date(item.CompletedDate);
    return (completedDate - activatedDate) / (1000 * 60 * 60 * 24);
  });
  
  // Calculate average and standard deviation
  const leadTimeAvg = leadTimes.reduce((sum, time) => sum + time, 0) / leadTimes.length;
  const cycleTimeAvg = cycleTimes.reduce((sum, time) => sum + time, 0) / cycleTimes.length;
  
  const leadTimeVariance = leadTimes.reduce((sum, time) => sum + Math.pow(time - leadTimeAvg, 2), 0) / leadTimes.length;
  const cycleTimeVariance = cycleTimes.reduce((sum, time) => sum + Math.pow(time - cycleTimeAvg, 2), 0) / cycleTimes.length;
  
  const leadTimeStdDev = Math.sqrt(leadTimeVariance);
  const cycleTimeStdDev = Math.sqrt(cycleTimeVariance);
  
  // Calculate coefficient of variation (lower is more predictable)
  const leadTimeCoV = leadTimeStdDev / leadTimeAvg;
  const cycleTimeCoV = cycleTimeStdDev / cycleTimeAvg;
  
  return {
    leadTime: {
      average: leadTimeAvg,
      stdDev: leadTimeStdDev,
      coefficientOfVariation: leadTimeCoV
    },
    cycleTime: {
      average: cycleTimeAvg,
      stdDev: cycleTimeStdDev,
      coefficientOfVariation: cycleTimeCoV
    }
  };
}
```
### Integrating with CI/CD Pipelines
You can integrate the dashboard deployment with your CI/CD pipelines to automate deployment when your template changes:

### Azure DevOps Pipeline Example
```yaml
trigger:
- main  # Or any branch where you store your dashboard templates

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: dashboard-deployment-vars  # Variable group containing PAT and other sensitive info

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '14.x'
  displayName: 'Install Node.js'

- script: |
    npm install axios
  displayName: 'Install Dependencies'

- script: |
    node deploy-dashboards.js
  displayName: 'Deploy Standard Dashboards'
  env:
    AZURE_DEVOPS_ORG: $(organization)
    AZURE_DEVOPS_PROJECT: $(project)
    AZURE_DEVOPS_PAT: $(pat)
```


## Conclusion
Configuring Lead Time and Cycle Time widgets in Azure DevOps using the REST API provides powerful flexibility for tracking team efficiency. By programmatically managing these metrics, you can create consistent dashboards across projects, automate dashboard updates, and build custom reporting solutions.
The four methods outlined in this blog provide options for different organizational needs:

1. **Process Template Method:** For formal, template-driven organizations
2. **Dashboard-as-Code Method:** For DevOps-oriented teams that prefer script-based approaches
3. **Azure DevOps Extension Method:** For large organizations with non-technical users
4. **Fully Automated Deployment with Event Triggers:** For enterprise environments requiring zero-touch standardization

By implementing these standardized metrics, your organization can:

- Identify bottlenecks across teams and projects
- Benchmark performance between teams
- Track improvement initiatives over time
- Set realistic expectations with stakeholders based on historical data

Remember that the ultimate goal is to use these metrics to drive continuous improvement. Regularly review your time metrics with your teams and implement strategies to address any issues that arise. By focusing on both Lead Time (for customer satisfaction) and Cycle Time (for team efficiency), you can optimize your entire delivery pipeline.
When choosing an implementation method, consider your organization's size, technical expertise, governance requirements, and frequency of team creation. The most sophisticated method—fully automated event-driven deployment—ensures perfect standardization but requires more initial setup. For smaller organizations or those just starting with these metrics, the Dashboard-as-Code method might provide a better balance of simplicity and flexibility.
Whichever method you choose, consistently tracking Lead Time and Cycle Time will provide valuable insights into your development process and help you deliver value to customers faster while maintaining a sustainable pace for your teams.

