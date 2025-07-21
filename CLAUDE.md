# MongoDB Atlas Stream Processing - Best Practices Guide

This document outlines proven best practices for implementing MongoDB Atlas Stream Processing pipelines efficiently.

Things you need from the user:
- orgId & projectId for use in atlas cli
- mongodb connection string
- userName & password for use in the connection string
- dbRole for use in the connection string
- cluster name

Reference:
- atlas cli: https://www.mongodb.com/docs/atlas/cli/current/
- atlas stream processing: https://www.mongodb.com/docs/atlas/atlas-stream-processing/

## 🎯 Key Success Factors

### 1. **Always Discover Schema First**
Before creating any pipeline, understand your data source structure. For schema discovery, DONOT write back to the source, create a atlas sink connection to write to:

```javascript
// Schema discovery pattern
const samplerName = 'schemaSampler';

// Create minimal processor
sp.createStreamProcessor(samplerName, [
  {
    $source: {
      connectionName: "your_data_source"
    }
  },
  {
    $merge: {
      into: {
        connectionName: "your_atlas_connection",
        db: "temp",
        coll: "sample"
      }
    }
  }
]);

// Start and sample
sp[samplerName].start();
let sampleDocs = sp[samplerName].sample();

// If no data yet, wait briefly
if (!sampleDocs || sampleDocs.length === 0) {
  sleep(1000);
  sampleDocs = sp[samplerName].sample();
}

// Stop and cleanup
sp[samplerName].stop();
sp[samplerName].drop();

// Analyze the schema
if (sampleDocs && sampleDocs.length > 0) {
  printjson(sampleDocs[0]);  // Shows full document structure
  analyzeSchema(sampleDocs[0]);  // Use the helper function below
}

#### Schema Analysis Helper:
```javascript
// Recursive schema analyzer for nested documents
const analyzeSchema = (obj, prefix = '') => {
  Object.keys(obj).forEach(key => {
    const value = obj[key];
    const path = prefix ? `${prefix}.${key}` : key;
    
    if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
      print(`${path}: object`);
      analyzeSchema(value, path);  // Recurse for nested objects
    } else {
      const type = Array.isArray(value) ? 'array' : typeof value;
      print(`${path}: ${type} = ${JSON.stringify(value)}`);
    }
  });
};
```

### 2. **Understand Connection Requirements**
List available connections before building pipelines:

```javascript
// Check available connections
sp.listConnections();
```

#### Creating Atlas Connections for Stream Processing
To create Atlas cluster connections for pipeline output:

**Step 1: Login to Atlas CLI**
```bash
atlas auth login
# Follow the browser authentication process
```

**Step 2: Find your stream processing instance**
```bash
atlas streams instances list --projectId <projectId>
```

**Step 3: Find your cluster name**
```bash
atlas clusters list --projectId <projectId>
```

**Step 4: Create connection configuration file**
```json
{
  "name": "your_connection_name", 
  "type": "Cluster",
  "clusterName": "YourClusterName",
  "dbRoleToExecute": {
    "role": "atlasAdmin",
    "type": "BUILT_IN"
  }
}
```

**Step 5: Create the connection**
```bash
atlas streams connections create your_connection_name \
  --instance your_instance_name \
  --file connection_config.json \
  --projectId <projectId>
```

**Step 6: Verify connection created**
```bash
atlas streams connections list \
  --instance your_instance_name \
  --projectId <projectId>
```

**Step 7: Delete connection when done**
```bash
atlas streams connections delete your_connection_name \
  --instance your_instance_name \
  --projectId <projectId> \
  --force
```

**Key Requirements:**
- Must include `dbRoleToExecute` with `role` and `type`
- Use `type: "Cluster"` for Atlas cluster connections
- Authenticate with `atlas auth login` first
- Connection names must be unique within the instance

### 3. **Pipeline Structure Rules**

#### ✅ **Required Pipeline Structure:**
```javascript
[
  { $source: { connectionName: "data_source" } },
  // Optional: Processing stages
  { $tumblingWindow: { /* windowing logic */ } },
  // Required: Must end with one of these
  { $merge: { into: { /* output destination */ } } }
  // OR { $emit: { /* emit destination */ } }
  // OR { $externalFunction: { /* external function */ } }
]
```

#### ❌ **Common Mistakes to Avoid:**
- Pipeline without sink stage (`$merge`, `$emit`, or `$externalFunction`)
- Using `$limit` outside of window pipeline
- Incorrect connection names in sink stages

### 4. **Windowing Best Practices**

For stateful operations (like `$group`, `$median`), use windowing:

```javascript
{
  $tumblingWindow: {
    interval: { size: 10, unit: "second" },
    pipeline: [
      {
        $group: {
          _id: "$group_field",
          max_value: { $max: "$field" },
          min_value: { $min: "$field" },
          avg_value: { $avg: "$field" },
          median_value: { 
            $median: {
              input: "$field",
              method: "approximate"
            }
          },
          count: { $sum: 1 },
          unique_items: { $addToSet: "$identifier" }
        }
      },
      {
        $addFields: {
          unique_count: { $size: "$unique_items" }
        }
      }
    ]
  }
}
```

### 5. **Output Structure Optimization**

Create clean, structured output documents:

```javascript
{
  $project: {
    _id: 0,  // Remove MongoDB _id
    group_id: "$_id",  // Rename grouping field
    stats: {
      max: "$max_value",
      min: "$min_value", 
      avg: { $round: ["$avg_value", 2] },  // Round decimals
      median: "$median_value"
    },
    metrics: {
      document_count: "$count",
      unique_devices: "$unique_count"
    },
    window: {
      start: {$meta: “stream.window.start”},
      end: {$meta: “stream.window.end”},
    }
  }
}
```

### 6. **Error Handling & Debugging**

#### Connection Authentication:
```bash
mongosh "mongodb://your-stream-url/" \
  --tls --authenticationDatabase admin \
  --username your_username --password your_password
```

#### Common Error Solutions:
- **"auth required"** → Add `--username` and `--password`
- **"last stage must be $merge"** → Add proper sink stage
- **"$limit only permitted in window"** → Move `$limit` inside window pipeline. Or don't use window
- **"Expected Atlas connection"** → Use correct connection name for sink

### 7. **Testing & Monitoring**

#### Create Processor:
```javascript
sp.createStreamProcessor('processorName', pipeline);
sp.processorName.start();
```

#### Monitor Performance:
```javascript
sp.processorName.stats();     // Get performance metrics
sp.processorName.sample();    // View output samples
```

#### Management Commands:
```javascript
sp.listStreamProcessors();    // List all processors
sp.processorName.stop();      // Stop processor
sp.processorName.drop();      // Delete processor
```
