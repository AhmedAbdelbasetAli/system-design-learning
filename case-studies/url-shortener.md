Designing a URL Shortener
A URL shortener is a service that takes a long URL and generates a shorter, unique alias (e.g., bit.ly/abc123). This is commonly used to make URLs more manageable, especially for sharing on social media or embedding in messages.

Table of Contents
Problem Statement
Functional Requirements
Non-Functional Requirements
High-Level Design
Database Design
Encoding Strategy
API Design
Scalability Considerations
Trade-offs
Optimizations
Real-World Examples
Problem Statement
The goal is to design a scalable and efficient URL shortening service. Users should be able to:

Provide a long URL and receive a shortened version.
Use the shortened URL to redirect to the original long URL.
The system must handle millions of requests per day while ensuring low latency and high availability.

Functional Requirements
Generate Short URL :
Accept a long URL as input and return a unique, shortened URL.
Redirect to Original URL :
When a user visits the shortened URL, redirect them to the original long URL.
Customizable Short URLs (Optional) :
Allow users to specify a custom alias for their shortened URL (e.g., bit.ly/my-custom-url).
Analytics (Optional) :
Track the number of clicks on each shortened URL.
Non-Functional Requirements
Scalability :
Handle millions of URLs and redirects per day.
Low Latency :
Redirects should occur in milliseconds.
High Availability :
Ensure the service remains operational even during peak usage.
Data Consistency :
Ensure the mapping between short and long URLs is accurate.
Security :
Protect against malicious URLs and abuse.
High-Level Design
Components:
Web Server :
Handles incoming HTTP requests for creating and redirecting URLs.
API Layer :
Exposes endpoints for generating short URLs and handling redirects.
Database :
Stores mappings between short and long URLs.
Cache :
Improves performance by storing frequently accessed mappings in memory.
Load Balancer :
Distributes traffic across multiple web servers.
Workflow:
Shorten URL :
User submits a long URL via an API.
The system generates a unique short URL and stores the mapping in the database.
The short URL is returned to the user.
Redirect :
User visits the short URL.
The system looks up the original long URL and redirects the user.
Database Design
Schema:

CREATE TABLE url_mappings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_url VARCHAR(10) NOT NULL UNIQUE,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    click_count INT DEFAULT 0
); 

Key Fields:
short_url: The unique identifier for the shortened URL.
long_url: The original long URL.
created_at: Timestamp when the mapping was created.
click_count: Tracks the number of times the short URL has been accessed.
Indexing:
Create an index on short_url to ensure fast lookups.
Encoding Strategy
To generate a short URL, we need a unique identifier. A common approach is to use Base62 encoding (a combination of uppercase letters, lowercase letters, and digits).

Steps:
Generate a Unique ID :
Use an auto-incrementing database ID or a distributed ID generator (e.g., Snowflake).
Encode the ID :
Convert the numeric ID into a Base62 string.
Example: ID 12345 → Short URL dnh.
Example Code (Python): 

import string

BASE62 = string.ascii_letters + string.digits  # "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

def encode_base62(num):
    if num == 0:
        return BASE62[0]
    encoded = []
    while num > 0:
        num, remainder = divmod(num, 62)
        encoded.append(BASE62[remainder])
    return ''.join(reversed(encoded))

API Design
Endpoints:
POST /shorten :
Request Body: { "long_url": "https://example.com/very-long-url" }
Response: { "short_url": "https://bit.ly/abc123" }
GET /{short_url} :
Redirects to the original long URL.
Example Implementation (Flask): 

from flask import Flask, request, redirect
from base62_encoder import encode_base62  # Assume this contains the Base62 encoder
import sqlite3

app = Flask(__name__)
db = sqlite3.connect("url_shortener.db", check_same_thread=False)

@app.route("/shorten", methods=["POST"])
def shorten():
    data = request.json
    long_url = data.get("long_url")
    cursor = db.cursor()
    cursor.execute("INSERT INTO url_mappings (long_url) VALUES (?)", (long_url,))
    db.commit()
    short_id = encode_base62(cursor.lastrowid)
    return {"short_url": f"https://bit.ly/{short_id}"}

@app.route("/<short_id>")
def redirect_to_long(short_id):
    cursor = db.cursor()
    cursor.execute("SELECT long_url FROM url_mappings WHERE short_url = ?", (short_id,))
    result = cursor.fetchone()
    if result:
        return redirect(result[0])
    return "URL not found", 404 

Scalability Considerations
Horizontal Scaling :
Deploy multiple instances of the web server behind a load balancer.
Caching :
Use Redis to cache frequently accessed mappings.
Sharding :
Partition the database by short URL prefixes (e.g., a-*, b-*) to distribute load.
Auto-Scaling :
Dynamically adjust resources based on traffic patterns.
Trade-offs
Custom Aliases vs. Random IDs :
Custom aliases provide flexibility but require additional checks for uniqueness.
Consistency vs. Performance :
Strong consistency ensures accurate mappings but may slow down writes.
Storage vs. Computation :
Precomputing short URLs simplifies redirection but increases storage requirements.
Optimizations
Batch Inserts :
Group multiple URL mappings into a single database write operation.
Prefetching :
Cache popular URLs proactively to reduce database queries.
Compression :
Compress long URLs before storing them to save space.
Real-World Examples
1. Bitly
Bitly uses a centralized database with caching to handle billions of redirects daily. It also provides analytics to track clicks and referrers.

2. TinyURL
TinyURL allows users to create custom aliases and supports expiration policies for short URLs.

Conclusion
Designing a URL shortener involves balancing simplicity, scalability, and performance. By using techniques like Base62 encoding, caching, and sharding, you can build a robust system capable of handling high traffic volumes. For further exploration, consider implementing custom features like expiration policies or analytics tracking.
