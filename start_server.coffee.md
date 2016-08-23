# Start server

## Core imports

	# fs = require 'fs'

	http = require 'http'


## Library imports

	bunyan = require 'bunyan'

	h = require 'hyperscript'

	{bind} = require 'ramda'

	redis = require 'redis'


## Relative imports

	request_handler_maker = require './server'


## Logging

	log = bunyan.createLogger
		level: 'trace'
		name: 'start_server'
		serializers: bunyan.stdSerializers
		# serializers:
			# err: bunyan.stdSerializers.err
			# req: bunyan.stdSerializers.req
			# res: bunyan.stdSerializers.res
		# serializers:
		# 	chunk: pick(['field'])
		# stream: fs.createWriteStream('server.log.json')
		# stream: fs.createWriteStream('start_server.log.json')

	# log_error = bind(log.error, log)
	# log_info = bind(log.info, log)
	log_trace = bind(log.trace, log)


## Redis client

	redis_client = redis.createClient()

	redis_client.on 'error', (err)->
		log.error err: err, 'Redis client threw an error.'


## Run!

	listener = request_handler_maker log, redis_client

	server = http.createServer listener

	hostname = 'localhost'

	port = 1337

	server.listen port, hostname, ->
		log.info "Started server listening on #{hostname}:#{port}."

	# redis_client.quit()
