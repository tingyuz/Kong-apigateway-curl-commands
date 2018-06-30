# Kong apigateway curl commands

Manul step to set up KONG docker container and initialization for EdgeX microservices.

#### create database for kong, using posrgres sql
docker run -d --name kong-database -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" postgres:9.5

#### start kong database docker container
docker run --rm --link kong-database:kong-database -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database"  kong:0.13.0 kong migrations up

#### start kong container
docker run -d --name kong --link kong-database:kong-database -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" -e "KONG_ADMIN_LISTEN_SSL=0.0.0.0:8444" -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 kong:0.13.0

#### use below to set up kong as the same user-defined network as edgex
docker network connect composefiles_edgex-network kong

#### need to use IP of local host (192.168.1.152) in the upstream_url when adding each api for the edgex microservice.

#### create service for each edgex microservice 
curl -i -X POST --url http://localhost:8001/services/ -d "name=coredata" -d "host=192.168.1.151" -d "port=48080" -d "protocol=http"

curl -i -X POST --url http://localhost:8001/services/ -d "name=metadata" -d "host=192.168.1.151" -d "port=48081" -d "protocol=http"

curl -i -X POST --url http://localhost:8001/services/ -d "name=command" -d "host=192.168.1.151" -d "port=48082" -d "protocol=http"

curl -i -X POST --url http://localhost:8001/services/ -d "name=notifications" -d "host=192.168.1.151" -d "port=48060" -d "protocol=http"

#### use below for testing x-custom-id which may be used later for our EdgeX micro service
curl -i -X POST --url http://localhost:8001/services/ -d "name=headertest" -d "host=192.168.1.151" -d "port=3000" -d "protocol=http"


#### create route for the service
curl -i -X POST --url http://localhost:8001/services/coredata/routes -d "paths[]=/coredata"

curl -i -X POST --url http://localhost:8001/services/metadata/routes -d "paths[]=/metadata"

curl -i -X POST --url http://localhost:8001/services/command/routes -d "paths[]=/command"

curl -i -X POST --url http://localhost:8001/services/notifications/routes -d "paths[]=/notifications"

curl -i -X POST --url http://localhost:8001/services/headertest/routes -d "paths[]=/headertest"

#### use below to access the microservice through proxy
curl -i -X GET --url http://localhost:8000/coredata/api/v1/ping

#### use below to access the headertest webservice
curl -i -X GET --url http://localhost:8000/headertest/headercheck

#### if use postman, we need to enable postman interceptor as header "host" is restricted and it will be blocked by default unless interceptor is enabled.

// currently we are using these partial uri to map the individual microservice of edgex
coredata - core data
metadata - metadata
command - command
notifications - support notifications

#### enable plugins ( auth, log etc ) for kong. we are using JWT here for api1 right now, so a request to coredata will be denied if there is no jwt.
curl -X POST http://localhost:8001/services/coredata/plugins -d "name=jwt" 

#### create a consumer "ibm" for at kong so that ibm can access coredata service
curl -X POST http://localhost:8001/consumers -d "username=ibm"
		
#### use below to create JWT credential after creating consumer ibm,
curl -X POST http://localhost:8001/consumers/ibm/jwt -H "Content-Type: application/x-www-form-urlencoded"

#### we can delete the jwt credential later with url below, id is what we got from the url above
curl -X DELETE http://localhost:8001/consumers/ibm}/jwt/{id}

#### use this to get the jwt token for the consumer 
curl -X GET http://localhost:8001/consumers/ibm/jwt

{"total":1,"data":[{"created_at":1523288175000,"id":"6918c367-5a77-4805-8f0b-f4ed646ab989","algorithm":"HS256","key":"G96QIZNIrXPXdw8m6iNBLpBd46W1gzsV","secret":"Da7gkA2CzRNa8V4iGk7pXKsVstoEi748","consumer_id":"863448a8-36d5-4a68-ae76-d40ac068fede"}]}

#### go to https://jwt.io/ and use information below to get JWT token. notice iss is the key we got when creating jwt token, and uncheck secret base64 encoded

header: 
{
  "typ": "JWT",
  "alg": "HS256"  
}

payload data
{
  "iss": "G96QIZNIrXPXdw8m6iNBLpBd46W1gzsV"
}

secret: Da7gkA2CzRNa8V4iGk7pXKsVstoEi748

#### use this below to access the resource 
curl -i -X GET --url http://localhost:8000/coredata/api/v1/ping -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJHOTZRSVpOSXJYUFhkdzhtNmlOQkxwQmQ0NlcxZ3pzViIsInVzZXJuYW1lIjoiaWJtIn0.Bkao_RCwbW9kUA8jhYj9OlqxxKK-VkC7MleEW6U9uuw" 

#### access resource with query string with jwt
curl -i -X GET --url http://localhost:8000/coredata/api/v1/ping?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJHOTZRSVpOSXJYUFhkdzhtNmlOQkxwQmQ0NlcxZ3pzViIsInVzZXJuYW1lIjoiaWJtIn0.Bkao_RCwbW9kUA8jhYj9OlqxxKK-VkC7MleEW6U9uuw

#### access resource with cookie with jwt. need to set config.cookie_names which is disabled by default
curl -i -X GET --url http://localhost:8000/coredata/api/v1/ping -b "jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJHOTZRSVpOSXJYUFhkdzhtNmlOQkxwQmQ0NlcxZ3pzViIsInVzZXJuYW1lIjoiaWJtIn0.Bkao_RCwbW9kUA8jhYj9OlqxxKK-VkC7MleEW6U9uuw"


////////////////////////////////
//config oAuth2 with KONG, this is not finished yet.
////////////////////////////////

curl -X POST http://localhost:8001/apis/api2/plugins -d "name=oauth2" -d  "config.scopes=email,phone,address" -d "config.mandatory_scope=true" -d  "config.enable_authorization_code=true"

//create a consumer 
curl -X POST http://localhost:8001/consumers/ -data "username=ibm"

// create an application
curl -X POST http://localhost:8001/consumers/ibm/oauth2 -d "name=TestApp" -d "redirect_uri=http://localhost:8000/metadata/api/v1/ping"

returns
{"client_id":"OSXdZ5Ga2oqlNUmECyJfyJdLRxU2gtxh","created_at":1522954954000,"id":"14469e5f-d48b-40d3-a651-bb11ba8531bf","redirect_uri":["http:\/\/localhost:8000\/metadata\/api\/v1\/ping"],"name":"TestApp","client_secret":"Khhp26r11TSKmQZCUoNtT0E6QilV14IW","consumer_id":"441762cf-9fb6-4e46-9219-85017f3a03f9"}


//get access token for the application
curl -X POST http://localhost:8001/oauth2_tokens -d "credential_id=14469e5f-d48b-40d3-a651-bb11ba8531bf" -d "expires_in=3600

{"token_type":"bearer","access_token":"fM1T2eiBEfoJLUyFLns1WPrWsxQFcHAI","created_at":1522956043000,"expires_in":3600,"credential_id":"14469e5f-d48b-40d3-a651-bb11ba8531bf","id":"95ae5891-d93d-428a-ae1f-aede21ada685"}
