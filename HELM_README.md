# Minimal Helm Chart for Go HTTP Server

This is a minimal Helm chart that templatizes only the sensitive data in your existing YAML files.

## 🎯 **What Changed (Minimal)**

- **Only templated sensitive data**: Database connection string and image details
- **Everything else remains the same**: Replicas, resources, ports, etc.
- **Moved files to templates folder**: For Helm to process them

## 📁 **Chart Structure**

```
my-app/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Only sensitive data
└── templates/              # Your existing YAML files
    ├── deployment.yaml     # Minimal templating (image only)
    ├── service.yaml        # No changes
    ├── configmap.yaml      # No changes
    └── secret.yaml         # Templated connection string
```

## 🚀 **Usage**

### **Install**
```bash
helm install my-app .
```

### **Install with custom values**
```bash
helm install my-app . --values values.yaml
```

### **Override specific values**
```bash
helm install my-app . \
  --set database.connectionString="postgresql://user:pass@host:5432/db" \
  --set image.tag="v1.0.0"
```

### **Upgrade**
```bash
helm upgrade my-app .
```

### **Uninstall**
```bash
helm uninstall my-app
```

## 🔧 **What You Can Customize**

Only these values are templated (everything else is hardcoded as before):

```yaml
# values.yaml
database:
  connectionString: "postgresql://username:password@localhost:5432/database_name"

image:
  repository: "kanavghai/my-app"
  tag: "latest"
```

## 📋 **Benefits**

- ✅ **Minimal changes** to your existing setup
- ✅ **Sensitive data templated** (no more hardcoded secrets)
- ✅ **Easy to customize** image and database connection
- ✅ **Everything else unchanged** (replicas, resources, ports)
- ✅ **Simple to use** - just `helm install my-app .`

## 🔍 **Verify**

```bash
# Check what will be deployed
helm install my-app . --dry-run --debug

# Check values being used
helm get values my-app
```

That's it! Minimal templating, maximum compatibility with your existing setup. 🎉
