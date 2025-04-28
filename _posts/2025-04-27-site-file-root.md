---
layout: post
title: "Site file root determination"
author: "Shtille"
categories: journal
tags: [thoughts,cpp,libmicrohttpd]
---

I found a way to determine site file root.

For example, we have a site with main content and a subsite.
We could check for a referer header value to find out if it's a root file request. Then we can add response cookie to make all subsequent headers have its value.

```cpp
// Lookup for "Referer" header value
const char* referer = MHD_lookup_connection_value(connection, MHD_HEADER_KIND, "Referer");
if (referer == nullptr)
{
	// No referer provided - this means we are at base file
	// Remember the resource we are using for referred requests
	char cookie_buffer[128];
	ResourceID resource_id = resource_->id();
	snprintf(cookie_buffer, sizeof(cookie_buffer) / sizeof(cookie_buffer[0]), 
		"%s=%u; SameSite=Strict", "resource_id", resource_id);
	AddHeader_SetCookie(cookie_buffer);
}
```

Then in main function we should check for a cookie value stored previously to make proper file responses.
```cpp
// At first check for referer
const char* referer = MHD_lookup_connection_value(connection, MHD_HEADER_KIND, "Referer");
if (referer != nullptr)
{
	// Resource cookie is set in FileResponse (file.cpp)
	// Cookie is just a resource ID number
	const char * cookie = MHD_lookup_connection_value(connection, MHD_COOKIE_KIND, "resource_id");
	if (cookie != nullptr)
	{
		// Resource ID exists - find resource by its ID and set as preferred
		wsk::ResourceID resource_id;
		sscanf(cookie, "%u", &resource_id);
		wsk::Resource * resource = server->FindResourceByID(resource_id);
		server->SetPreferredResource(resource);
	}
	else
		server->SetPreferredResource(nullptr);
}
else
{
	// No referer - clear preferred resource
	server->SetPreferredResource(nullptr);
}
```