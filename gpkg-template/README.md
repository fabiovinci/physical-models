# geoIACS GeoPackage template

This folder contains the [geoIACS GeoPackage template](geoIACS_template_2.0.gpkg)


# GeoPackage Database Structure (geoIACS 2.0)

This document describes the data structure of the GeoPackage (GPKG) file for the geoIACS 2.0 model. It details the architectural choices made to translate the hierarchical XML/GML schema (XSD) into a relational database model suitable for spatial analysis in standard GIS environments.

## Translating the XSD Model to GeoPackage: Key Design Choices

The transition from an XSD schema to a GeoPackage requires adapting hierarchical object-oriented concepts into a flat, relational table structure. The most significant architectural decisions revolve around the handling of spatial data.

### The Challenge of Polymorphic Geometries
In the geoIACS XSD model, features are defined with specific spatial properties:
*   **Strict Surfaces:** Elements like `ReferenceParcel`, `AgriculturalParcel`, `AgriculturalArea`, and `OtherEligibleArea` are defined with `gml:SurfacePropertyType` [1-4]. These naturally translate into standard polygon tables.
*   **Flexible Geometries:** Elements like `EcoLandscapeElement`, `Site`, and `EcologicalFocusArea` are defined with `gml:GeometryPropertyType` [5-7], meaning a single conceptual feature can be represented as a Point, a LineString, or a Polygon.

Standard relational databases and GIS software (such as QGIS) operate on the principle of **"one geometry type per layer."** While SQLite can technically store mixed geometries in a single column, doing so breaks compatibility with most GIS clients, which require homogeneous spatial tables for rendering and indexing. 

### The Split-Table Strategy
To solve the polymorphic geometry challenge and ensure full GIS compatibility, features with flexible geometries have been explicitly **split into separate tables** based on their spatial type (e.g., one table for points, one for lines, one for polygons). 

Depending on the complexity of the feature, two different relational strategies were applied to manage these split tables:

1.  **Normalized Structure with Views (Applied to EcoLandscapeElement):** 
    Because `EcoLandscapeElement` features share many complex attributes and relationships (like the many-to-many link with Reference Parcels), their data was normalized. A central non-spatial table (`ecolandscapeelement`) holds all alphanumeric attributes [8, 9]. The spatial data is isolated in three "child" tables (`ecolandscapeelementpoint`, `ecolandscapeelementlinestring`, `ecolandscapeelementpolygon`) [10-13]. Finally, SQL `VIEW`s were created to merge the central attributes with the specific geometries on the fly, providing ready-to-use layers for QGIS [14-17].
2.  **Flat Structure (Applied to Site and EcologicalFocusArea):**
    For features with fewer interconnected relationships, a "flat" approach was used. The attributes defined in the XSD are physically repeated across three independent spatial tables (e.g., `sitepoint`, `sitelinestring`, `sitepolygon` [18-20] and `ecologicalfocusareapoint`, `ecologicalfocusarealinestring`, `ecologicalfocusareapolygon` [21-23]). This provides immediate, standalone layers without the need for complex joins.

---

## Entity-Relationship (ER) Diagram

The following diagram illustrates the logical relationships between the entities in the GeoPackage, reflecting the conceptual links defined in the XSD model.

```mermaid
erDiagram
    %% Core Geographic Pillar
    REFERENCEPARCEL {
        INTEGER id PK
        TEXT rpid 
        Polygon geometry
    }

    AGRICULTURALPARCEL {
        INTEGER id PK
        TEXT apid 
        TEXT relatedrpid FK
        Polygon geometry
    }

    AGRICULTURALAREA {
        INTEGER id PK
        TEXT aaid 
        TEXT relatedrpid FK
        Polygon geometry
    }

    OTHERELIGIBLEAREA {
        INTEGER id PK
        TEXT oeaid 
        TEXT relatedrpid FK
        Polygon geometry
    }

    %% EcoLandscapeElement (Normalized Strategy)
    ECOLANDSCAPEELEMENT {
        INTEGER id PK
        TEXT eleid 
        TEXT eletype_href
    }

    ELE_GEOMETRY_TABLES {
        TEXT eleid FK
        Geometry geometry
    }

    ECOLANDSCAPEELEMENT_REFERENCEPARCEL {
        INTEGER id PK
        TEXT eleid FK
        TEXT rpid FK
    }

    %% Site and Farm Animals (Flat Strategy)
    SITE_GEOMETRY_TABLES {
        TEXT siteid 
        Geometry geometry
    }

    FARMANIMALSPECIES {
        INTEGER id PK
        TEXT siteid FK
    }

    NACEACTIVITYVALUE {
        INTEGER id PK
        TEXT siteid FK
    }

    %% Ecological Focus Area (Flat Strategy)
    ECOLOGICALFOCUSAREA_GEOMETRY_TABLES {
        TEXT efaid 
        Geometry geometry
    }

    %% Logical Relationships
    REFERENCEPARCEL ||--o{ AGRICULTURALPARCEL : "contains (relatedRP)"
    REFERENCEPARCEL ||--o{ AGRICULTURALAREA : "contains (relatedRP)"
    REFERENCEPARCEL ||--o{ OTHERELIGIBLEAREA : "contains (relatedRP)"
    
    REFERENCEPARCEL ||--o{ ECOLANDSCAPEELEMENT_REFERENCEPARCEL : "associated_via"
    ECOLANDSCAPEELEMENT ||--o{ ECOLANDSCAPEELEMENT_REFERENCEPARCEL : "associated_via"
    
    ECOLANDSCAPEELEMENT ||--o| ELE_GEOMETRY_TABLES : "has_geometry (Point/Line/Poly)"
    SITE_GEOMETRY_TABLES ||--o{ FARMANIMALSPECIES : "hosts"
    SITE_GEOMETRY_TABLES ||--o{ NACEACTIVITYVALUE : "has_activity"
