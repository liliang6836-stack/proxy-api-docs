# proxy-api-docs

Developer documentation and API examples for reliable proxy infrastructure.

## C++ libcurl proxy example

This example fetches the same HTTPS endpoint through an HTTP proxy and a SOCKS5
proxy. Set `PROXY_USERPWD` to `username:password` in the environment when the
proxy requires authentication. Do not commit proxy credentials to source
control.

```cpp
#include <curl/curl.h>

#include <cstdlib>
#include <iostream>
#include <string>

static size_t write_response(char* data, size_t size, size_t count, void* user_data)
{
	if (size != 0 && count > static_cast<size_t>(-1) / size)
		return 0;

	const size_t bytes = size * count;
	try
	{
		static_cast<std::string*>(user_data)->append(data, bytes);
	}
	catch (...)
	{
		return 0;
	}
	return bytes;
}

static CURLcode fetch_url(const char* url, const char* proxy,
						  curl_proxytype proxy_type, std::string& response)
{
	CURL* curl = curl_easy_init();
	if (curl == nullptr)
		return CURLE_FAILED_INIT;

	response.clear();
	curl_easy_setopt(curl, CURLOPT_URL, url);
	curl_easy_setopt(curl, CURLOPT_PROXY, proxy);
	curl_easy_setopt(curl, CURLOPT_PROXYTYPE, proxy_type);
	curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_response);
	curl_easy_setopt(curl, CURLOPT_WRITEDATA, &response);
	curl_easy_setopt(curl, CURLOPT_LOW_SPEED_TIME, 10L);
	curl_easy_setopt(curl, CURLOPT_LOW_SPEED_LIMIT, 50L);
	curl_easy_setopt(curl, CURLOPT_MAX_RECV_SPEED_LARGE,
					 static_cast<curl_off_t>(2000000));

	if (const char* credentials = std::getenv("PROXY_USERPWD"))
		curl_easy_setopt(curl, CURLOPT_PROXYUSERPWD, credentials);

	const CURLcode result = curl_easy_perform(curl);
	curl_easy_cleanup(curl);
	return result;
}

int main()
{
	const CURLcode init_result = curl_global_init(CURL_GLOBAL_DEFAULT);
	if (init_result != CURLE_OK)
	{
		std::cerr << "curl_global_init: " << curl_easy_strerror(init_result) << '\n';
		return 1;
	}

	const char* url = "https://ipinfo.io/json";
	const char* proxy = "global.9http.com:9091";
	std::string response;

	const curl_proxytype proxy_types[] = {CURLPROXY_HTTP, CURLPROXY_SOCKS5_HOSTNAME};
	const char* proxy_names[] = {"HTTP", "SOCKS5"};
	int exit_code = 0;

	for (size_t index = 0; index < 2; ++index)
	{
		const CURLcode result = fetch_url(url, proxy, proxy_types[index], response);
		if (result == CURLE_OK)
			std::cout << proxy_names[index] << " response: " << response << '\n';
		else
		{
			std::cerr << proxy_names[index] << " request failed: "
					  << curl_easy_strerror(result) << '\n';
			exit_code = 1;
		}
	}

	curl_global_cleanup();
	return exit_code;
}
```

Link against libcurl when building. For example, with a compiler and libcurl
development package that provide `pkg-config`:

```sh
c++ -std=c++11 proxy_example.cpp $(pkg-config --cflags --libs libcurl) -o proxy_example
```

`CURLPROXY_SOCKS5_HOSTNAME` asks the proxy to resolve the destination hostname.
Use `CURLPROXY_SOCKS5` instead if DNS resolution should happen locally. The
response is stored in a dynamically growing `std::string`, so the caller does
not need to guess a fixed buffer size.
