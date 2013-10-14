This is the README for Libredis (http://github.com/toymachine/libredis)

Libredis is general client library for Redis written in C.

A PHP extension is also included based on this lower-level C library.

The goal of Libredis is to provide efficient low-level plumbing for accessing Redis servers.

It should be easy to build higher level abstractions on top of Libredis
(of which the included PHP extension is an example).

Libredis provides a fast and efficient IO implementation for talking concurrently
with multiple Redis servers (using non-blocking asynchronous IO).
It uses a 'batch' abstraction in order to support pipeline-ing
(e.g. send more than 1 command to Redis in a single operation), and
finally a consistent hashing implementation (Ketama) is provided for partitioning keys across servers.

The low-level library knows about and validates any possible Redis result, but does not restrict you
in what commands you send trough it, e.g. you can/must format commands yourself. This allows the library
to be used with any new and upcoming Redis versions without updates, as long as the
format of the results does not change.

As a convenience, The PHP extension does at least support formatting the get and set commands, or arbitrary
commands using the 'cmd' method.

The C API is documented in include/redis.h and an example is given in test.c (make c_test)

Examples in PHP (in increasing order of complexity)

-- Hello world (in PHP, using the included php extension) --

$libredis = Libredis();
$connection = $libredis->get_connection("127.0.0.1");
//set a key:
$connection->set('hello', 'world', 200); //set the key hello to 'world' with a timeout of 200ms
//get a key:
echo $connection->get('hello', 200); //get the value of key hello with a timeout of 200ms

-- Multiple commands at the same time --

//multi-set
$batch = $libredis->create_batch();
$batch->set('hello', 'world');
$batch->set('hello2', 'world2';
$batch->execute($connection, 200); //same as $connection->execute($batch, ...)
//multi-get
$batch = $libredis->create_batch();
$batch->get('hello');
$batch->get('hello2');
$connection->execute($batch, 200); //same as $batch->execute($connection, ...)
while($batch->next_reply($reply_type, $reply_value, $reply_length)) {
    echo $reply_value, PHP_EOL;
}

-- Writing any command (using multi-bulk command protocol) --

$connection = $libredis->get_connection("127.0.0.1:6379");
$batch = $libredis->create_batch();
$batch->cmd("MGET", "key1", "key2", "anotherOne");
$batch->cmd("HINCRBY", "aKey", "aField", 10);
$connection->execute($batch);
...

-- Writing command explicitly (incr in this example) --
$connection = $libredis->get_connection("127.0.0.1:6379");
$batch = $libredis->create_batch();
$batch->write("INCR incr_test\r\n", 1);
$connection->execute($batch);
while($level = $batch->next_reply($reply_type, $reply_value, $reply_length)) {
    echo "start", PHP_EOL;
    echo "\ttype ", $reply_type, PHP_EOL;
    echo "\tval ", $reply_value, " (", gettype($reply_value), ")", PHP_EOL;
    echo "\tlen ", $reply_length, PHP_EOL;
    echo "\tlevel ", $level, PHP_EOL;
    echo "end", PHP_EOL;
}

-- Talking to multiple servers in parallel --

$libredis = Libredis();
$connection1 = $libredis->get_connection("127.0.0.1:6379");
$connection2 = $libredis->get_connection("127.0.0.1:6380");
$batch1 = $libredis->create_batch();
$batch1->set('hello', 'world'); //add a 'set' command to batch1
$batch2 = $libredis->create_batch();
$batch2->set('hello2', 'world2');
$executor = $libredis->create_executor();
$executor->add($connection1, $batch1); //associate connection1 with batch1
$executor->add($connection2, $batch2); //associate connection2 with batch2
//execute all batches against their corresponding connections in parallel, with a
//500 ms timeout on the whole operation.
$executor->execute(500);

So basically,

1. Commands are written into Batches; Batch::cmd(...), Batch::set(key, value), Batch::get(key), Batch::write(...)
2. Batches are associated with Connections trough an Executor; Executor::add(connection, batch)
3. The total operation is performed within a configurable timeout by calling Executor::execute(timeout)
4. When execution returns, batches will contain replies that can be read out; Batch::next_reply(...)

For simple cases (single server, single batch), batches also can be executed directly; Batch::execute(connection, timeout)

An implementation of a multi-server multi-get can be found in test.php (test_mget)
