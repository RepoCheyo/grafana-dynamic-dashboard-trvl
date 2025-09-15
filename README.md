# 📊 Grafana + MySQL on Docker

This guide shows how to run **MySQL** and **Grafana** containers with Docker, connect them together, and create dynamic dashboards with variables.

---

## 🚀 1. Create a Docker network

Grafana and MySQL need to communicate. Put them in the same network:

```bash
docker network create monitoring-net
```

---

## 🗄️ 2. Run MySQL container

```bash
docker run -d \
  --name mysql-grafana \
  --network monitoring-net \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=monitoring \
  -e MYSQL_USER=grafana \
  -e MYSQL_PASSWORD=grafana123 \
  -p 3306:3306 \
  mysql:8.0
```

- **Name:** `mysql-grafana`
- **Database:** `monitoring`
- **User:** `grafana` / `grafana123`
- **Port:** exposed on `3306` for host access

---

## 📊 3. Run Grafana container

```bash
docker run -d \
  --name grafana \
  --network monitoring-net \
  -p 3000:3000 \
  grafana/grafana:latest
```

- **Name:** `grafana`
- **Web UI:** [http://localhost:3000](http://localhost:3000)
- **Default login:** `admin / admin` (you'll be asked to change password)

---

## 🔗 4. Connect Grafana to MySQL

1. Open Grafana UI → **Configuration** → **Data Sources** → **Add data source**.
2. Select **MySQL**.
3. Fill in:
   - **Host:** `mysql-grafana:3306` (because they share a Docker network)
   - **Database:** `monitoring`
   - **User:** `grafana`
   - **Password:** `grafana123`
4. Click **Save & Test** → it should connect successfully.

---

## 📦 5. Seed monitoring tables

Enter the MySQL CLI:

```bash
docker exec -it mysql-grafana mysql -u grafana -pgrafana123 monitoring
```

Create sample tables for the dashboard examples:

```sql
-- CPU usage monitoring table
CREATE TABLE cpu_usage (
  id INT AUTO_INCREMENT PRIMARY KEY,
  hostname VARCHAR(50),
  usage_percent DECIMAL(5,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data with different timestamps
INSERT INTO cpu_usage (hostname, usage_percent, created_at) VALUES
('server-1', 20.1, '2025-09-14 04:00:00'),
('server-1', 35.4, '2025-09-14 04:05:00'),
('server-1', 55.6, '2025-09-14 04:10:00'),
('server-2', 25.3, '2025-09-14 04:00:00'),
('server-2', 42.1, '2025-09-14 04:05:00'),
('server-2', 38.7, '2025-09-14 04:10:00'),
('server-3', 18.9, '2025-09-14 04:00:00'),
('server-3', 31.2, '2025-09-14 04:05:00'),
('server-3', 44.8, '2025-09-14 04:10:00');
```

---

## 🎛️ 6. Setting Up Dynamic Dashboard Variables

Variables make your dashboards dynamic and reusable. Here's how to create them:

### Creating a Server Variable

1. Go to **Dashboard Settings** → **Variables** → **Add variable**
2. Configure the variable (see `image7.png` in images folder for reference):
   - **Variable type:** Query
   - **Name:** `server`
   - **Label:** `Unit`
   - **Data source:** Select your MySQL data source
   - **Query:**
     ```sql
     SELECT 'server-1'
     UNION
     SELECT 'server-2'
     UNION
     SELECT 'server-3'
     ```

This creates a dropdown allowing users to select which server to monitor.

---

## 📈 7. Creating Dashboard Panels

### Panel 1: Time Series Chart

_Reference: `image1.png` in images folder_

1. **Add Panel** → **Time series**
2. **Query:**
   ```sql
   SELECT
     created_at as time,
     usage_percent,
     created_at
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ORDER BY created_at
   ```
3. **Panel Options:**
   - Title: "Stream usage history"
   - Legend: Show as table

### Panel 2: Bar Chart

_Reference: `image2.png` in images folder_

1. **Add Panel** → **Bar chart**
2. **Query:**
   ```sql
   SELECT
     created_at as time,
     usage_percent
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ORDER BY created_at
   ```
3. **Panel Options:**
   - Title: "Usage history"
   - Orientation: Auto
   - Show values: Always

### Panel 3: Stat Panel (Count)

_Reference: `image5.png` in images folder_

1. **Add Panel** → **Stat**
2. **Query:**
   ```sql
   SELECT COUNT(*)
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ```
3. **Panel Options:**
   - Title: "Total number of records"
   - Color mode: Value
   - Text alignment: Auto

### Panel 4: Gauge (Max Usage)

_Reference: `image4.png` in images folder_

1. **Add Panel** → **Gauge**
2. **Query:**
   ```sql
   SELECT MAX(usage_percent)
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ```
3. **Panel Options:**
   - Title: "Max registered % of usage"
   - Min: 0, Max: 100
   - Thresholds: Green (0-50), Red (50-100)

### Panel 5: Gauge (Min Usage)

_Reference: `image6.png` in images folder_

1. **Add Panel** → **Gauge**
2. **Query:**
   ```sql
   SELECT MIN(usage_percent)
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ```
3. **Panel Options:**
   - Title: "Min registered % of usage"
   - Min: 0, Max: 100
   - Thresholds: Green (0-50), Red (50-100)

---

## 🔧 8. Advanced Query Techniques

### Using Time Range Variables

Grafana provides built-in time range variables:

- `$__timeFrom()` - Start of time range
- `$__timeTo()` - End of time range
- `$__timeFilter(column)` - Complete time filter

Example query with time filtering:

```sql
SELECT
  created_at as time,
  usage_percent
FROM monitoring.cpu_usage
WHERE $__timeFilter(created_at)
  AND hostname = '$server'
ORDER BY created_at
```

### Using Multiple Variables

Create additional variables for more dynamic dashboards:

1. **Metric Variable:**

   ```sql
   SELECT 'usage_percent' as __text, 'usage_percent' as __value
   UNION
   SELECT 'CPU Load' as __text, 'cpu_load' as __value
   ```

2. **Query using multiple variables:**
   ```sql
   SELECT
     created_at as time,
     $metric
   FROM monitoring.cpu_usage
   WHERE hostname = '$server'
   ORDER BY created_at
   ```

---

## 📊 9. Dashboard Layout Tips

### Multi-Panel Layout

_Reference: `image3.png` in images folder shows a complete dashboard_

- Arrange panels in a logical flow
- Use the same time range for all panels
- Group related metrics together
- Use consistent color schemes

### Panel Sizing

- **Full width:** Time series charts work best full width
- **Half width:** Gauges and stats work well in pairs
- **Quarter width:** Simple metrics can be grouped in rows of 4

---

## 🎨 10. Styling and Customization

### Color Schemes

- Use thresholds for status indication (green/yellow/red)
- Maintain consistency across panels
- Consider accessibility with color choices

### Units and Formatting

- Set appropriate units (%, MB, seconds, etc.)
- Use decimal places consistently
- Format large numbers with appropriate prefixes

---

## 🔄 11. Refresh and Auto-Update

Set up automatic dashboard refresh:

1. **Dashboard Settings** → **Time range**
2. Set **Auto refresh:** 5s, 10s, 30s, 1m, etc.
3. Configure **Refresh intervals** based on your data update frequency

---

## 🛑 12. Stop and clean up

Stop containers:

```bash
docker stop grafana mysql-grafana
```

Remove containers:

```bash
docker rm grafana mysql-grafana
```

Remove network:

```bash
docker network rm monitoring-net
```

---

## 📝 13. Docker Compose Alternative

Create a `docker-compose.yml` for easier management:

```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-grafana
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: monitoring
      MYSQL_USER: grafana
      MYSQL_PASSWORD: grafana123
    ports:
      - '3306:3306'
    networks:
      - monitoring-net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - '3000:3000'
    networks:
      - monitoring-net
    depends_on:
      - mysql

networks:
  monitoring-net:
    driver: bridge
```

Run with: `docker-compose up -d`

---

✅ You now have a complete Grafana + MySQL setup with dynamic dashboards and variables!

**Image References:**

- `image1.png` - Time series visualization panel
- `image2.png` - Bar chart configuration
- `image3.png` - Complete dashboard layout
- `image4.png` - Max usage gauge panel
- `image5.png` - Record count stat panel
- `image6.png` - Min usage gauge panel
- `image7.png` - Variable configuration screen

All referenced images are located in the `images/` folder.
