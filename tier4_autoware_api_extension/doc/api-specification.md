# tier4_autoware_api_extension API Specification

This document describes the external API endpoints provided by `tier4_autoware_api_extension`.

## Overview

This package provides external API adapters that bridge between internal Autoware topics/services and external API interfaces. All endpoints follow the pattern `/api/external/{get|set}/...`.

## Input Validation

All API endpoints implement input validation:

| Communication Type | Invalid Input Behavior |
|-------------------|----------------------|
| **Topic** | Log warning + clamp to valid range |
| **Service** | Return `ResponseStatus::ERROR` with error message |

---

## VelocityLimit

Manages velocity limits for the planning system.

### Service: `/api/external/set/velocity_limit`

Sets the maximum velocity for autonomous driving.

**Type**: `tier4_external_api_msgs/srv/SetVelocityLimit`

**Request**:
| Field | Type | Unit | Range | Description |
|-------|------|------|-------|-------------|
| `velocity` | float32 | m/s | >= 0.0, finite | Target maximum velocity |

**Response**:
| Field | Type | Description |
|-------|------|-------------|
| `status.code` | uint16 | `SUCCESS` (1) or `ERROR` (3) |
| `status.message` | string | Error description (empty on success) |

**Validation**:
- Returns `ERROR` if velocity is negative, NaN, or Inf

### Topic: `/api/external/get/velocity_limit`

Publishes the current velocity limit.

**Type**: `tier4_external_api_msgs/msg/VelocityLimit`

**Fields**:
| Field | Type | Unit | Range | Description |
|-------|------|------|-------|-------------|
| `stamp` | builtin_interfaces/Time | - | - | Timestamp |
| `velocity` | float32 | m/s | >= 0.0 | Current max velocity (clamped) |

**QoS**: Transient Local, Depth 1

---

## TrafficLight

Provides nearest traffic light information.

### Topic: `/api/external/get/nearest_traffic_light_group`

Publishes the nearest traffic light group state.

**Type**: `tier4_external_api_msgs/msg/TrafficLightGroup`

**Fields**:
| Field | Type | Description |
|-------|------|-------------|
| `traffic_light_group_id` | int64 | Traffic light group identifier |
| `elements` | TrafficLightElement[] | List of traffic light elements |

**TrafficLightElement Fields**:
| Field | Type | Valid Values | Description |
|-------|------|--------------|-------------|
| `color` | uint8 | 0-4 | UNKNOWN(0), RED(1), AMBER(2), GREEN(3), WHITE(4) |
| `shape` | uint8 | 0-10 | UNKNOWN(0), CIRCLE(1), arrows(2-9), CROSS(10) |
| `status` | uint8 | 0-3 | UNKNOWN(0), SOLID_OFF(1), SOLID_ON(2), FLASHING(3) |
| `confidence` | float32 | [0.0, 1.0] | Detection confidence (clamped) |

**Validation**:
- Invalid enum values are set to UNKNOWN (0)
- Confidence is clamped to [0.0, 1.0]

---

## RouteDistance

Provides remaining distance to goal.

### Topic: `/api/external/get/route_distance`

Publishes the remaining distance along the route.

**Type**: `tier4_external_api_msgs/msg/RouteDistance`

**Fields**:
| Field | Type | Unit | Range | Description |
|-------|------|------|-------|-------------|
| `stamp` | builtin_interfaces/Time | - | - | Timestamp |
| `remaining_distance` | float64 | m | >= 0.0 | Distance to goal (clamped) |

**Validation**:
- Negative values are clamped to 0.0
- NaN/Inf values are clamped to 0.0

---

## PlanningFactor

Aggregates planning factors from multiple planning modules.

### Topic: `/api/external/get/planning_factors`

Publishes aggregated planning factors.

**Type**: `tier4_external_api_msgs/msg/PlanningFactorArray`

**Fields**:
| Field | Type | Description |
|-------|------|-------------|
| `stamp` | builtin_interfaces/Time | Aggregation timestamp |
| `factors` | PlanningFactor[] | List of planning factors |

**PlanningFactor Fields**:
| Field | Type | Valid Values | Description |
|-------|------|--------------|-------------|
| `header` | std_msgs/Header | - | Message header |
| `behavior_type` | uint16 | 0-7 | UNKNOWN(0), NONE(1), SLOW_DOWN(2), STOP(3), SHIFT_LEFT(4), SHIFT_RIGHT(5), TURN_LEFT(6), TURN_RIGHT(7) |
| `behavior_name` | string | - | Module name |
| `behavior_detail` | string | - | Detailed description |
| `control_points` | PlanningFactorControlPoint[] | - | Control points |
| `is_safe` | bool | - | Safety status |
| `decision_detail` | string | - | Decision description |
| `object_factors` | DecisionFactorObject[] | - | Object-based factors |
| `point_cloud_factors` | DecisionFactorPointCloud[] | - | Point cloud-based factors |

**PlanningFactorControlPoint Fields**:
| Field | Type | Unit | Range | Description |
|-------|------|------|-------|-------------|
| `pose` | geometry_msgs/Pose | - | - | Control point pose |
| `distance` | float32 | m | finite | Distance from ego (clamped to 0 if NaN/Inf) |

**Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `rate` | double | 10.0 | Publishing rate (Hz) |
| `timeout` | double | 1.0 | Message validity duration (s) |
| `topics` | string[] | - | List of input topics |

**Validation**:
- Invalid behavior types are set to UNKNOWN (0)
- NaN/Inf distances are set to 0.0
- Stale messages (older than timeout) are removed

---

## Error Handling

### ResponseStatus Codes

| Code | Name | Description |
|------|------|-------------|
| 1 | SUCCESS | Request completed successfully |
| 3 | ERROR | Request failed due to invalid input |

### Common Error Messages

| Error | Description |
|-------|-------------|
| "velocity is not finite (NaN or Inf)" | Velocity value is NaN or Infinity |
| "velocity must be non-negative" | Velocity value is negative |

---

## Usage Examples

### Setting Velocity Limit (ROS 2 CLI)

```bash
ros2 service call /api/external/set/velocity_limit \
  tier4_external_api_msgs/srv/SetVelocityLimit \
  "{velocity: 10.0}"
```

### Subscribing to Route Distance

```bash
ros2 topic echo /api/external/get/route_distance
```

### Subscribing to Planning Factors

```bash
ros2 topic echo /api/external/get/planning_factors
```
