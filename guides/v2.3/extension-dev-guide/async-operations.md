---
group: php-developer-guide
title: Asynchronous and deferred operations
---

Asynchronous operations are not native to PHP but it is still possible to execute heavy
operations simultaneously, or delay them until they absolutely have to be finished.

To make writing asynchronous code easier, Magento provides the `DeferredInterface` to use with asynchronous operations.
This allows client code to work with asynchronous operations just as it would with standard operations.

## DeferredInterface

`Magento\Framework\Async\DeferredInterface` is quite simple:

```php
interface DeferredInterface
{
    /**
     * @return mixed Value.
     * @throws \Throwable
     */
    public function get();

    public function isDone(): bool;
}
```

When the client code needs the result, the `get()` method will be called to retrieve the result.
`isDone()` can be used to see whether the code has completed.

There are 2 types of asynchronous operations where `DeferredInterface` can be used to describe the result:

* With asynchronous operations in progress, calling `get()` would wait for them to finish and return their result.
* With deferred operations, `get()` would actually start the operation, wait for it to finish, and then return the result.

Sometimes developers require more control over long asynchronous operations.
That is why there is an extended deferred variant - `Magento\Framework\Async\CancelableDeferredInterface`:

```php
interface CancelableDeferredInterface extends DeferredInterface
{
    /**
     * @param bool $force Cancel operation even if it's already started.
     * @return void
     * @throws CancelingDeferredException When failed to cancel.
     */
    public function cancel(bool $force = false): void;

    /**
     * @return bool
     */
    public function isCancelled(): bool;
}
```

This interface is for operations that may take too long and can be canceled.

### Client code

Assuming that `serviceA`, `serviceB` and `serviceC` all execute asynchronous operations, such as HTTP requests, the client code would look like:

```php
public function aMethod() {
    //Started executing 1st operation
    $operationA = $serviceA->executeOp();

    //Executing 2nd operations at the same time
    $operationB = $serviceB->executeOp2();

    //We need to wait for 1st operation to start operation #3
    $serviceC->executeOp3($operationA->get());

    //We don't have to wait for operation #2, let client code wait for it if it needs the result
    //Operation number #3 is being executed simultaneously with operation #2
    return $operationB;
}
```

And not a callback in sight!

With the deferred client, the code can start multiple operations at the same time, wait for operations required to finish and pass the promise of a result to another method.

## ProxyDeferredFactory

When writing a module or an extension, you may not want to burden other developers with having to know that your method is performing an asynchronous operation.
There is a way to hide it: employ the autogenerated factory `YourClassName\ProxyDeferredFactory`. With its help, you can return values that seem like regular objects
but are in fact deferred results.

For example:

```php
public function __construct(CallResult\ProxyDeferredFactory $callResultFactory)
{
    $this->proxyDeferredFactory = $callResultFactory;
}

....

public function doARemoteCall(string $uniqueValue): CallResult
{
    //Async HTTP request, get() will return a CallResult instance.
    //Call is in progress.
    $deferredResult = $this->client->call($uniqueValue);

    //Returns CallResult instance that will call $deferredResult->get() when any of the object's methods is used.
    return $this->proxyDeferredFactory->create(['deferred' => $deferredResult]);
}

public function doCallsAndProcess(): Result
{
    //Both calls running simultaneously
    $call1 = $this->doARemoteCall('call1');
    $call2 = $this->doARemoteCall('call2');

    //Only when CallResult::getStuff() is called the $deferredResult->get() is called.
    return new Result([
        'call1' => $call1->getStuff(),
        'call2' => $call2->getStuff()
    ]);
}
```

## Using DeferredInterface for background operations

As mentioned above, the first type of asynchronous operations are operations executing in a background.
`DeferredInterface` can be used to give client code a promise of a not-yet-received result and wait for it by calling the `get()` method.

Take a look at an example: creating shipments for multiple products:

```php
class DeferredShipment implements DeferredInterface
{
    private $request;

    private $done = false;

    private $trackingNumber;

    public function __construct(AsyncRequest $request)
    {
        $this->request = $request;
    }

    public function isDone() : bool
    {
        return $this->done;
    }

    public function get()
    {
        if (!$this->trackingNumber) {
            $this->request->wait();
            $this->trackingNumber = json_decode($this->request->getBody(), true)['tracking'];

            $this->done = true;
        }

        return $this->trackingNumber;
    }
}

class Shipping
{
    ....

    public function ship(array $products): array
    {
        $shipments = [];
        //Shipping simultaneously
        foreach ($products as $product) {
            $shipments[] = new DeferredShipment(
                $this->client->sendAsync(['id' => $product->getId()])
            );
        }

        return $shipments;
    }
}

class ShipController
{
    ....

    public function execute(Request $request): Response
    {
        $shipments = $this->shipping->ship($this->products->find($request->getParam('ids')));
        $trackingsNumbers = [];
        foreach ($shipments as $shipment) {
            $trackingsNumbers[] = $shipment->get();
        }

        return new Response(['trackings' => $trackingNumbers]);
    }
}
```

Here, multiple shipment requests are being sent at the same time with their results gathered later.
If you do not want to write your own `DeferredInterface` implementation, you can use `CallbackDeferred` to provide callbacks that will be used when `get()` is called.

## Using DeferredInterface for deferred operations

The second type of asynchronous operations are operations that are being postponed and executed only when a result is absolutely needed.

An example:

Assume you are creating a repository for an entity and you have a method that returns a singular entity by ID.
You want to make a performance optimization for cases when multiple entities are requested during the same request-response process, so you would not load them separately.

```php
class EntityRepository
{
    private $requestedEntityIds = [];

    private $identityMap = [];

    ...

    /**
     * @return Entity[]
     */
    public function findMultiple(array $ids): array
    {
        .....

        //Adding found entities to the identity map be able to find them by ID.
        foreach ($found as $entity) {
            $this->identityMap[$entity->getId()] = $entity;
        }

        ....
    }

    public function find(string $id): Entity
    {
        //Adding this ID to the list of previously requested IDs.
        $this->requestedEntityIds[] = $id;

        //Returning deferred that will find all requested entities
        //and return the one with $id
        return $this->proxyDefferedFactory->createFor(
            Entity::class,
            new CallbackDeferred(
                function () use ($id) {
                    if (empty($this->identityMap[$id])) {
                        $this->findMultiple($this->requestedEntityIds);
                        $this->requestedEntityIds = [];
                    }

                    return $this->identityMap[$id];
                }
            )
        );
    }

    ....
}

class EntitiesController
{
    ....

    public function execute(): Response
    {
        //No actual DB query issued
        $criteria1Id = $this->entityService->getEntityIdWithCriteria1();
        $criteria2Id = $this->entityService->getEntityIdWithCriteria2();
        $criteria1Entity = $this->entityRepo->find($criteria1Id);
        $criteria2Entity = $this->entityRepo->find($criteria2Id);

        //Querying the DB for both entities only when getStringValue() is called the 1st time.
        return new Response(
            [
                'criteria1' => $criteria1Entity->getStringValue(),
                'criteria2' => $criteria2Entity->getStringValue()
            ]
        );
    }
}
```

## Examples in Magento

Please see our asynchronous HTTP client `Magento\Framework\HTTP\AsyncClientInterface` and `Magento\Shipping\Model\Shipping` with various `Magento\Shipping\Model\Carrier\AbstractCarrierOnline` implementations to see how `DeferredInterface` can be used to work with asynchronous code.
