# Итоговые замены (все 12 замечаний)

Применять к репозиторию, который вы приложили. Перемещать и удалять ничего не нужно. Всего 21 файл.

## Код (`src/Lab1`)

**1. `src/Lab1/Inventory/Goods.cs`** — новое содержимое целиком:

```csharp
namespace Itmo.ObjectOrientedProgramming.Lab1.Inventory;

public sealed class Goods
{
    private readonly List<LineItem> _lineItems;

    public Goods()
    {
        _lineItems = new List<LineItem>();
    }

    public Goods(IEnumerable<LineItem> lineItems)
    {
        _lineItems = lineItems
            .GroupBy(item => item.Sku, item => item.Quantity)
            .Select(group => new LineItem(group.Key, group.Sum()))
            .ToList();
    }

    public IReadOnlyCollection<LineItem> LineItems => _lineItems.AsReadOnly();

    public VolumetricWeight VolumetricWeight => new VolumetricWeight(
        _lineItems.Sum(item => item.VolumetricWeight.Value));

    public bool IsEmpty => _lineItems.Count == 0;

    public int CountUnits(Sku sku)
    {
        return _lineItems
            .Where(item => item.Sku.Equals(sku))
            .Sum(item => item.Quantity);
    }

    public bool Includes(Goods goods)
    {
        return goods.LineItems.All(item => CountUnits(item.Sku) >= item.Quantity);
    }

    public Goods Add(LineItem item)
    {
        return new Goods(_lineItems.Append(item));
    }

    public Goods Merge(Goods goods)
    {
        return new Goods(_lineItems.Concat(goods.LineItems));
    }

    public Goods Subtract(Goods goods)
    {
        if (Includes(goods) is false)
        {
            throw new ArgumentException("Subtracted goods are not included", nameof(goods));
        }

        IEnumerable<LineItem> rest = _lineItems
            .Where(item => item.Quantity > goods.CountUnits(item.Sku))
            .Select(item => item.Reduce(goods.CountUnits(item.Sku)));

        return new Goods(rest);
    }
}
```

**2. `src/Lab1/Inventory/LineItem.cs`** — новое содержимое целиком:

```csharp
namespace Itmo.ObjectOrientedProgramming.Lab1.Inventory;

public sealed record LineItem
{
    public LineItem(Sku sku, int quantity)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);

        Sku = sku;
        Quantity = quantity;
    }

    public Sku Sku { get; }

    public int Quantity { get; }

    public VolumetricWeight VolumetricWeight => Sku.VolumetricWeight.MultiplyBy(Quantity);

    public LineItem Reduce(int quantity)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(quantity);

        return new LineItem(Sku, Quantity - quantity);
    }
}
```

**3. `src/Lab1/Inventory/StorageSpace.cs`** — новое содержимое целиком:

```csharp
namespace Itmo.ObjectOrientedProgramming.Lab1.Inventory;

public sealed class StorageSpace
{
    public StorageSpace(VolumetricWeight capacity, Goods contents)
    {
        ArgumentOutOfRangeException.ThrowIfZero(capacity.Value, nameof(capacity));

        if (contents.VolumetricWeight > capacity)
        {
            throw new ArgumentException("Contents do not fit into the capacity", nameof(contents));
        }

        Capacity = capacity;
        Contents = contents;
    }

    public VolumetricWeight Capacity { get; }

    public Goods Contents { get; private set; }

    public PutResult Put(Goods goods)
    {
        VolumetricWeight freeCapacity = Capacity - Contents.VolumetricWeight;

        if (goods.VolumetricWeight > freeCapacity)
        {
            return new PutResult.Failure("Not enough free capacity");
        }

        Contents = Contents.Merge(goods);

        return new PutResult.Success();
    }

    public TakeResult Take(Goods goods)
    {
        if (Contents.Includes(goods) is false)
        {
            return new TakeResult.Failure("Some of the goods are missing");
        }

        Contents = Contents.Subtract(goods);

        return new TakeResult.Success();
    }
}
```

**4. `src/Lab1/Inventory/VolumetricWeight.cs`** — новое содержимое целиком:

```csharp
namespace Itmo.ObjectOrientedProgramming.Lab1.Inventory;

public sealed record VolumetricWeight
{
    public VolumetricWeight(decimal value)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(value);

        Value = value;
    }

    public decimal Value { get; }

    // static нужен: в C# операторы бывают только статическими, и a - b читается лучше метода.
    public static VolumetricWeight operator -(VolumetricWeight left, VolumetricWeight right)
    {
        return new VolumetricWeight(left.Value - right.Value);
    }

    // static нужен: в C# операторы бывают только статическими, и a > b читается лучше метода.
    public static bool operator >(VolumetricWeight left, VolumetricWeight right)
    {
        return left.Value > right.Value;
    }

    // static нужен: в C# операторы только статические, а < обязателен в паре с >.
    public static bool operator <(VolumetricWeight left, VolumetricWeight right)
    {
        return left.Value < right.Value;
    }

    public VolumetricWeight MultiplyBy(int count)
    {
        return new VolumetricWeight(Value * count);
    }

    public int CountFitting(VolumetricWeight itemWeight)
    {
        ArgumentOutOfRangeException.ThrowIfZero(itemWeight.Value, nameof(itemWeight));

        decimal count = Math.Floor(Value / itemWeight.Value);

        return decimal.ToInt32(count);
    }
}
```

**5. `src/Lab1/Itineraries/DispatchStep.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using System.Diagnostics;

namespace Itmo.ObjectOrientedProgramming.Lab1.Itineraries;

public sealed class DispatchStep : IItineraryStep
{
    private readonly Warehouse _warehouse;
    private readonly Goods _manifest;

    public DispatchStep(Warehouse warehouse, Goods manifest)
    {
        if (manifest.IsEmpty)
        {
            throw new ArgumentException("Dispatch manifest must not be empty", nameof(manifest));
        }

        _warehouse = warehouse;
        _manifest = manifest;
    }

    public StepResult Perform(Truck truck)
    {
        if (_warehouse.IsAtDock(truck.Location) is false)
        {
            return new StepResult.Failure("Truck is not at the warehouse dock");
        }

        if (truck.Load(_manifest) is LoadResult.Failure loadingFailure)
        {
            return new StepResult.Failure($"Truck cannot load: {loadingFailure.Reason}");
        }

        return _warehouse.Dispatch(_manifest) switch
        {
            DispatchResult.Success dispatched => new StepResult.Success(dispatched.Duration),
            DispatchResult.Failure failure => RollBackLoading(truck, failure.Reason),
            _ => throw new UnreachableException(),
        };
    }

    private StepResult.Failure RollBackLoading(Truck truck, string reason)
    {
        if (truck.Unload(_manifest) is UnloadResult.Failure failure)
        {
            return new StepResult.Failure($"Rollback failed: {failure.Reason}");
        }

        return new StepResult.Failure($"Warehouse cannot dispatch: {reason}");
    }
}
```

**6. `src/Lab1/Itineraries/DriveStep.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;

namespace Itmo.ObjectOrientedProgramming.Lab1.Itineraries;

public sealed class DriveStep : IItineraryStep
{
    private readonly Location _destination;

    public DriveStep(Location destination)
    {
        _destination = destination;
    }

    public StepResult Perform(Truck truck)
    {
        TimeSpan travelTime = truck.DriveTo(_destination);

        return new StepResult.Success(travelTime);
    }
}
```

**7. `src/Lab1/Itineraries/Itinerary.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using System.Diagnostics;

namespace Itmo.ObjectOrientedProgramming.Lab1.Itineraries;

public sealed class Itinerary
{
    private readonly List<IItineraryStep> _steps;

    public Itinerary(IReadOnlyCollection<IItineraryStep> steps)
    {
        if (steps.Count == 0)
        {
            throw new ArgumentException("Itinerary must have at least one step", nameof(steps));
        }

        _steps = steps.ToList();
    }

    public ItineraryResult Simulate(Truck truck)
    {
        ItineraryResult start = new ItineraryResult.Success(TimeSpan.Zero);

        return _steps.Aggregate(start, (result, step) => PerformNext(result, step, truck));
    }

    private ItineraryResult PerformNext(ItineraryResult result, IItineraryStep step, Truck truck)
    {
        if (result is not ItineraryResult.Success success)
        {
            return result;
        }

        return step.Perform(truck) switch
        {
            StepResult.Success performed => new ItineraryResult.Success(
                success.TotalDuration + performed.Duration),
            StepResult.Failure failure => new ItineraryResult.Failure(failure.Reason),
            _ => throw new UnreachableException(),
        };
    }
}
```

**8. `src/Lab1/Itineraries/ItineraryResult.cs`** — новое содержимое целиком:

```csharp
namespace Itmo.ObjectOrientedProgramming.Lab1.Itineraries;

public abstract record ItineraryResult
{
    private ItineraryResult() { }

    public sealed record Success : ItineraryResult
    {
        public Success(TimeSpan totalDuration)
        {
            ArgumentOutOfRangeException.ThrowIfLessThan(totalDuration, TimeSpan.Zero);

            TotalDuration = totalDuration;
        }

        public TimeSpan TotalDuration { get; }
    }

    public sealed record Failure : ItineraryResult
    {
        public Failure(string reason)
        {
            ArgumentException.ThrowIfNullOrWhiteSpace(reason);

            Reason = reason;
        }

        public string Reason { get; }
    }
}
```

**9. `src/Lab1/Itineraries/ReceivingStep.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using System.Diagnostics;

namespace Itmo.ObjectOrientedProgramming.Lab1.Itineraries;

public sealed class ReceivingStep : IItineraryStep
{
    private readonly Warehouse _warehouse;
    private readonly Goods _manifest;

    public ReceivingStep(Warehouse warehouse, Goods manifest)
    {
        if (manifest.IsEmpty)
        {
            throw new ArgumentException("Receiving manifest must not be empty", nameof(manifest));
        }

        _warehouse = warehouse;
        _manifest = manifest;
    }

    public StepResult Perform(Truck truck)
    {
        if (_warehouse.IsAtDock(truck.Location) is false)
        {
            return new StepResult.Failure("Truck is not at the warehouse dock");
        }

        if (truck.Unload(_manifest) is UnloadResult.Failure unloadingFailure)
        {
            return new StepResult.Failure($"Truck cannot unload: {unloadingFailure.Reason}");
        }

        return _warehouse.Receive(_manifest) switch
        {
            ReceivingResult.Success received => new StepResult.Success(received.Duration),
            ReceivingResult.Failure failure => RollBackUnloading(truck, failure.Reason),
            _ => throw new UnreachableException(),
        };
    }

    private StepResult.Failure RollBackUnloading(Truck truck, string reason)
    {
        if (truck.Load(_manifest) is LoadResult.Failure failure)
        {
            return new StepResult.Failure($"Rollback failed: {failure.Reason}");
        }

        return new StepResult.Failure($"Warehouse cannot receive: {reason}");
    }
}
```

**10. `src/Lab1/Transport/Truck.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;

namespace Itmo.ObjectOrientedProgramming.Lab1.Transport;

public sealed class Truck
{
    private readonly StorageSpace _cargoHold;
    private readonly Speed _speed;

    public Truck(VolumetricWeight capacity, Speed speed, Location location)
    {
        _speed = speed;
        Location = location;
        _cargoHold = new StorageSpace(capacity, new Goods());
    }

    public VolumetricWeight Capacity => _cargoHold.Capacity;

    public Location Location { get; private set; }

    public Goods Cargo => _cargoHold.Contents;

    public LoadResult Load(Goods goods)
    {
        if (_cargoHold.Put(goods) is PutResult.Failure failure)
        {
            return new LoadResult.Failure(failure.Reason);
        }

        return new LoadResult.Success();
    }

    public UnloadResult Unload(Goods goods)
    {
        if (_cargoHold.Take(goods) is TakeResult.Failure failure)
        {
            return new UnloadResult.Failure(failure.Reason);
        }

        return new UnloadResult.Success();
    }

    public TimeSpan DriveTo(Location destination)
    {
        Distance distance = Location.DistanceTo(destination);
        Location = destination;

        return _speed.EstimateTravelTime(distance);
    }
}
```

**11. `src/Lab1/Warehousing/Crew.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;

namespace Itmo.ObjectOrientedProgramming.Lab1.Warehousing;

public sealed class Crew
{
    private readonly List<Employee> _employees;

    public Crew(IReadOnlyCollection<Employee> employees)
    {
        if (employees.Count == 0)
        {
            throw new ArgumentException("Crew must have at least one employee", nameof(employees));
        }

        _employees = employees.ToList();
    }

    public PlanningResult Plan(Goods goods)
    {
        if (goods.IsEmpty)
        {
            throw new ArgumentException("Goods must not be empty", nameof(goods));
        }

        if (CanHandle(goods) is false)
        {
            return new PlanningResult.Failure("Crew cannot lift some of the goods");
        }

        List<Round> rounds = SplitIntoRounds(goods);
        Round lastRound = rounds.Last();

        TimeSpan dispatchTime = rounds.Aggregate(
            TimeSpan.Zero,
            (total, round) => total + round.Duration);

        TimeSpan receivingTime = dispatchTime - lastRound.Duration + lastRound.HandlingDuration;

        return new PlanningResult.Success(rounds.Count, dispatchTime, receivingTime);
    }

    private bool CanHandle(Goods goods)
    {
        return goods.LineItems.All(item => CanLift(item.Sku));
    }

    private bool CanLift(Sku sku)
    {
        return _employees.Any(employee => employee.CanLift(sku));
    }

    private List<Round> SplitIntoRounds(Goods goods)
    {
        var rounds = new List<Round>();
        Goods rest = goods;

        while (rest.IsEmpty is false)
        {
            Round round = FormRound(rest);
            rounds.Add(round);
            rest = rest.Subtract(round.CarriedGoods);
        }

        return rounds;
    }

    private Round FormRound(Goods goods)
    {
        return _employees.Aggregate(
            new Round(),
            (round, employee) => round.Assign(employee, goods));
    }
}
```

**12. `src/Lab1/Warehousing/Employee.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;

namespace Itmo.ObjectOrientedProgramming.Lab1.Warehousing;

public sealed class Employee
{
    private readonly VolumetricWeight _maxLoad;

    public Employee(VolumetricWeight maxLoad, TimeSpan handlingTime, TimeSpan carryingTime)
    {
        ArgumentOutOfRangeException.ThrowIfZero(maxLoad.Value, nameof(maxLoad));
        ArgumentOutOfRangeException.ThrowIfLessThanOrEqual(handlingTime, TimeSpan.Zero);
        ArgumentOutOfRangeException.ThrowIfLessThanOrEqual(carryingTime, TimeSpan.Zero);

        _maxLoad = maxLoad;
        HandlingTime = handlingTime;
        RoundTime = handlingTime + carryingTime;
    }

    public TimeSpan HandlingTime { get; }

    public TimeSpan RoundTime { get; }

    public bool CanLift(Sku sku)
    {
        return sku.VolumetricWeight > _maxLoad is false;
    }

    public Goods PickUp(Goods goods)
    {
        return goods.LineItems.Aggregate(new Goods(), PickUpFitting);
    }

    private Goods PickUpFitting(Goods pickedUp, LineItem item)
    {
        VolumetricWeight freeLoad = _maxLoad - pickedUp.VolumetricWeight;
        int quantity = Math.Min(item.Quantity, freeLoad.CountFitting(item.Sku.VolumetricWeight));

        if (quantity == 0)
        {
            return pickedUp;
        }

        return pickedUp.Add(new LineItem(item.Sku, quantity));
    }
}
```

**13. `src/Lab1/Warehousing/Warehouse.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using System.Diagnostics;

namespace Itmo.ObjectOrientedProgramming.Lab1.Warehousing;

public sealed class Warehouse
{
    private const double DockRadiusInMeters = 10;

    private readonly StorageSpace _storage;
    private readonly Crew _crew;

    public Warehouse(
        WarehouseId id,
        Location location,
        VolumetricWeight capacity,
        Crew crew,
        Goods stock)
    {
        Id = id;
        Location = location;
        _crew = crew;
        _storage = new StorageSpace(capacity, stock);
    }

    public WarehouseId Id { get; }

    public Location Location { get; }

    public VolumetricWeight Capacity => _storage.Capacity;

    public Goods Stock => _storage.Contents;

    public bool IsAtDock(Location location)
    {
        return Location.DistanceTo(location).Meters <= DockRadiusInMeters;
    }

    public ReceivingResult Receive(Goods goods)
    {
        return _crew.Plan(goods) switch
        {
            PlanningResult.Success plan => PutIntoStorage(goods, plan),
            PlanningResult.Failure failure => new ReceivingResult.Failure(failure.Reason),
            _ => throw new UnreachableException(),
        };
    }

    public DispatchResult Dispatch(Goods goods)
    {
        return _crew.Plan(goods) switch
        {
            PlanningResult.Success plan => TakeFromStorage(goods, plan),
            PlanningResult.Failure failure => new DispatchResult.Failure(failure.Reason),
            _ => throw new UnreachableException(),
        };
    }

    private ReceivingResult PutIntoStorage(Goods goods, PlanningResult.Success plan)
    {
        if (_storage.Put(goods) is PutResult.Failure failure)
        {
            return new ReceivingResult.Failure(failure.Reason);
        }

        return new ReceivingResult.Success(plan.ReceivingTime, plan.RoundsCount);
    }

    private DispatchResult TakeFromStorage(Goods goods, PlanningResult.Success plan)
    {
        if (_storage.Take(goods) is TakeResult.Failure failure)
        {
            return new DispatchResult.Failure(failure.Reason);
        }

        return new DispatchResult.Success(plan.DispatchTime, plan.RoundsCount);
    }
}
```

## Тесты (`tests/Lab1.Tests`)

**14. `tests/Lab1.Tests/Itineraries/DispatchStepTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Itineraries;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Itineraries;

public class DispatchStepTests
{
    private readonly Sku _crate;
    private readonly Location _dockLocation;
    private readonly Employee _quickEmployee;
    private readonly Employee _strongEmployee;

    public DispatchStepTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));
        _dockLocation = new Location(59.90008, 30.3);

        _quickEmployee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        _strongEmployee = new Employee(
            new VolumetricWeight(18),
            TimeSpan.FromMinutes(3),
            TimeSpan.FromMinutes(7));
    }

    [Fact]
    public void Constructor_EmptyManifest_ThrowsArgumentException()
    {
        
        var employees = new List<Employee> { _quickEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 20);

        
        Exception? exception = Record.Exception(() => new DispatchStep(warehouse, new Goods()));

        
        Assert.IsType<ArgumentException>(exception);
    }

    [Fact]
    public void Perform_TruckAtDockAndGoodsInStock_MovesGoodsIntoTruck()
    {
        
        var truck = new Truck(new VolumetricWeight(60), new Speed(70), _dockLocation);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 20);
        var step = new DispatchStep(warehouse, CreateCrates(15));

        
        StepResult result = step.Perform(truck);

        
        StepResult.Success success = Assert.IsType<StepResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(20), success.Duration);
        Assert.Equal(15, truck.Cargo.CountUnits(_crate));
        Assert.Equal(5, warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Perform_OnlyStrongEmployee_ReturnsTimeOfThreeRounds()
    {
        
        var truck = new Truck(new VolumetricWeight(60), new Speed(70), _dockLocation);
        var employees = new List<Employee> { _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 20);
        var step = new DispatchStep(warehouse, CreateCrates(15));

        
        StepResult result = step.Perform(truck);

        
        StepResult.Success success = Assert.IsType<StepResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(30), success.Duration);
    }

    [Fact]
    public void Perform_StockMissesManifestGoods_ReturnsFailureAndLeavesTruckEmpty()
    {
        
        var truck = new Truck(new VolumetricWeight(60), new Speed(70), _dockLocation);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 10);
        var step = new DispatchStep(warehouse, CreateCrates(15));

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Empty(truck.Cargo.LineItems);
        Assert.Equal(10, warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Perform_TruckWithoutFreeCapacity_ReturnsFailureAndKeepsStock()
    {
        
        var truck = new Truck(new VolumetricWeight(30), new Speed(70), _dockLocation);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 20);
        var step = new DispatchStep(warehouse, CreateCrates(15));

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Empty(truck.Cargo.LineItems);
        Assert.Equal(20, warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Perform_TruckFartherThanTenMeters_ReturnsFailureAndKeepsStock()
    {
        
        var farLocation = new Location(59.9001, 30.3);
        var truck = new Truck(new VolumetricWeight(60), new Speed(70), farLocation);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, 20);
        var step = new DispatchStep(warehouse, CreateCrates(15));

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Empty(truck.Cargo.LineItems);
        Assert.Equal(20, warehouse.Stock.CountUnits(_crate));
    }

    private Warehouse CreateWarehouse(IReadOnlyCollection<Employee> employees, int cratesInStock)
    {
        return new Warehouse(
            new WarehouseId(3),
            new Location(59.9, 30.3),
            new VolumetricWeight(90),
            new Crew(employees),
            CreateCrates(cratesInStock));
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```

**15. `tests/Lab1.Tests/Itineraries/DriveStepTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Itineraries;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Itineraries;

public class DriveStepTests
{
    [Fact]
    public void Perform_TwoDegreesNorth_MovesTruckAndReturnsTravelTime()
    {
        
        var destination = new Location(57, 37);
        var truck = new Truck(new VolumetricWeight(40), new Speed(80), new Location(55, 37));
        var step = new DriveStep(destination);

        
        StepResult result = step.Perform(truck);

        
        StepResult.Success success = Assert.IsType<StepResult.Success>(result);
        Assert.Equal(destination, truck.Location);
        Assert.Equal(166.79, success.Duration.TotalMinutes, 2);
    }

    [Fact]
    public void Perform_SameLocation_TakesNoTime()
    {
        
        var location = new Location(55, 37);
        var truck = new Truck(new VolumetricWeight(40), new Speed(80), location);
        var step = new DriveStep(location);

        
        StepResult result = step.Perform(truck);

        
        StepResult.Success success = Assert.IsType<StepResult.Success>(result);
        Assert.Equal(TimeSpan.Zero, success.Duration);
    }
}
```

**16. `tests/Lab1.Tests/Itineraries/ItineraryTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Itineraries;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Itineraries;

public class ItineraryTests
{
    private readonly Sku _crate;
    private readonly Goods _manifest;
    private readonly Warehouse _source;
    private readonly Warehouse _destination;

    public ItineraryTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));
        _manifest = CreateCrates(15);

        var quickEmployee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        var strongEmployee = new Employee(
            new VolumetricWeight(18),
            TimeSpan.FromMinutes(3),
            TimeSpan.FromMinutes(7));

        _source = new Warehouse(
            new WarehouseId(1),
            new Location(59, 30),
            new VolumetricWeight(90),
            new Crew(new List<Employee> { quickEmployee, strongEmployee }),
            CreateCrates(20));

        _destination = new Warehouse(
            new WarehouseId(2),
            new Location(60, 30),
            new VolumetricWeight(90),
            new Crew(new List<Employee> { quickEmployee, strongEmployee }),
            new Goods());
    }

    [Fact]
    public void Simulate_DispatchDriveReceive_ReturnsSumOfStepDurations()
    {
        
        var truck = new Truck(new VolumetricWeight(60), new Speed(60), _source.Location);

        var itinerary = new Itinerary(new List<IItineraryStep>
        {
            new DispatchStep(_source, _manifest),
            new DriveStep(_destination.Location),
            new ReceivingStep(_destination, _manifest),
        });

        
        ItineraryResult result = itinerary.Simulate(truck);

        
        ItineraryResult.Success success = Assert.IsType<ItineraryResult.Success>(result);
        Assert.Equal(20 + 111.19 + 13, success.TotalDuration.TotalMinutes, 2);
        Assert.Equal(_destination.Location, truck.Location);
        Assert.Empty(truck.Cargo.LineItems);
        Assert.Equal(5, _source.Stock.CountUnits(_crate));
        Assert.Equal(15, _destination.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Simulate_ReceivingBeforeDriving_StopsAtFailedStep()
    {
        
        var truck = new Truck(new VolumetricWeight(60), new Speed(60), _source.Location);

        var itinerary = new Itinerary(new List<IItineraryStep>
        {
            new DispatchStep(_source, _manifest),
            new ReceivingStep(_destination, _manifest),
            new DriveStep(_destination.Location),
        });

        
        ItineraryResult result = itinerary.Simulate(truck);

        
        Assert.IsType<ItineraryResult.Failure>(result);
        Assert.Equal(_source.Location, truck.Location);
        Assert.Equal(15, truck.Cargo.CountUnits(_crate));
        Assert.Empty(_destination.Stock.LineItems);
    }

    [Fact]
    public void Simulate_TruckTooSmallForManifest_StopsAtDispatch()
    {
        
        var smallTruck = new Truck(new VolumetricWeight(30), new Speed(60), _source.Location);

        var itinerary = new Itinerary(new List<IItineraryStep>
        {
            new DispatchStep(_source, _manifest),
            new DriveStep(_destination.Location),
        });

        
        ItineraryResult result = itinerary.Simulate(smallTruck);

        
        Assert.IsType<ItineraryResult.Failure>(result);
        Assert.Equal(_source.Location, smallTruck.Location);
        Assert.Equal(20, _source.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Simulate_ItineraryAlreadyUsedBySlowTruck_TakesFastTruckSpeedIntoAccount()
    {
        
        var slowTruck = new Truck(new VolumetricWeight(60), new Speed(60), _source.Location);
        var fastTruck = new Truck(new VolumetricWeight(60), new Speed(120), _source.Location);
        Goods tenCrates = CreateCrates(10);

        var itinerary = new Itinerary(new List<IItineraryStep>
        {
            new DispatchStep(_source, tenCrates),
            new DriveStep(_destination.Location),
            new ReceivingStep(_destination, tenCrates),
        });

        itinerary.Simulate(slowTruck);

        
        ItineraryResult result = itinerary.Simulate(fastTruck);

        
        ItineraryResult.Success success = Assert.IsType<ItineraryResult.Success>(result);
        Assert.Equal(10 + 55.60 + 3, success.TotalDuration.TotalMinutes, 2);
        Assert.Equal(_destination.Location, fastTruck.Location);
        Assert.Equal(20, _destination.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Constructor_NoSteps_ThrowsArgumentException()
    {
        
        var noSteps = new List<IItineraryStep>();

        
        Exception? exception = Record.Exception(() => new Itinerary(noSteps));

        
        Assert.IsType<ArgumentException>(exception);
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```

**17. `tests/Lab1.Tests/Itineraries/ReceivingStepTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Itineraries;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Itineraries;

public class ReceivingStepTests
{
    private readonly Sku _crate;
    private readonly Location _dockLocation;

    public ReceivingStepTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));
        _dockLocation = new Location(59.90008, 30.3);
    }

    [Fact]
    public void Constructor_EmptyManifest_ThrowsArgumentException()
    {
        
        Warehouse warehouse = CreateWarehouse(new VolumetricWeight(60));

        
        Exception? exception = Record.Exception(() => new ReceivingStep(warehouse, new Goods()));

        
        Assert.IsType<ArgumentException>(exception);
    }

    [Fact]
    public void Perform_TruckAtDockWithManifestGoods_MovesGoodsWithoutLastCarrying()
    {
        
        Goods manifest = CreateCrates(8);
        Truck truck = CreateTruck(_dockLocation, manifest);
        Warehouse warehouse = CreateWarehouse(new VolumetricWeight(60));
        var step = new ReceivingStep(warehouse, manifest);

        
        StepResult result = step.Perform(truck);

        
        StepResult.Success success = Assert.IsType<StepResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(8), success.Duration);
        Assert.Empty(truck.Cargo.LineItems);
        Assert.Equal(8, warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Perform_TruckMissesManifestGoods_ReturnsFailureAndKeepsCargo()
    {
        
        Truck truck = CreateTruck(_dockLocation, CreateCrates(5));
        Warehouse warehouse = CreateWarehouse(new VolumetricWeight(60));
        var step = new ReceivingStep(warehouse, CreateCrates(8));

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Equal(5, truck.Cargo.CountUnits(_crate));
        Assert.Empty(warehouse.Stock.LineItems);
    }

    [Fact]
    public void Perform_WarehouseWithoutFreeCapacity_ReturnsFailureAndReturnsCargoToTruck()
    {
        
        Goods manifest = CreateCrates(8);
        Truck truck = CreateTruck(_dockLocation, manifest);
        Warehouse warehouse = CreateWarehouse(new VolumetricWeight(15));
        var step = new ReceivingStep(warehouse, manifest);

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Equal(8, truck.Cargo.CountUnits(_crate));
        Assert.Empty(warehouse.Stock.LineItems);
    }

    [Fact]
    public void Perform_TruckFartherThanTenMeters_ReturnsFailureAndKeepsCargo()
    {
        
        var farLocation = new Location(59.9001, 30.3);
        Goods manifest = CreateCrates(8);
        Truck truck = CreateTruck(farLocation, manifest);
        Warehouse warehouse = CreateWarehouse(new VolumetricWeight(60));
        var step = new ReceivingStep(warehouse, manifest);

        
        StepResult result = step.Perform(truck);

        
        Assert.IsType<StepResult.Failure>(result);
        Assert.Equal(8, truck.Cargo.CountUnits(_crate));
        Assert.Empty(warehouse.Stock.LineItems);
    }

    private Truck CreateTruck(Location location, Goods cargo)
    {
        var truck = new Truck(new VolumetricWeight(45), new Speed(70), location);
        truck.Load(cargo);

        return truck;
    }

    private Warehouse CreateWarehouse(VolumetricWeight capacity)
    {
        var employee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        return new Warehouse(
            new WarehouseId(2),
            new Location(59.9, 30.3),
            capacity,
            new Crew(new List<Employee> { employee }),
            new Goods());
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```

**18. `tests/Lab1.Tests/Transport/TruckTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Transport;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Transport;

public class TruckTests
{
    private readonly Sku _crate;
    private readonly Sku _barrel;
    private readonly Truck _truck;

    public TruckTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));
        _barrel = new Sku(new SkuId(102), "Barrel", new VolumetricWeight(5));
        _truck = new Truck(new VolumetricWeight(30), new Speed(60), new Location(59, 30));
    }

    [Theory]
    [InlineData(0.0)]
    [InlineData(-60.0)]
    [InlineData(double.NaN)]
    [InlineData(double.PositiveInfinity)]
    public void Constructor_InvalidSpeed_ThrowsOutOfRange(double kilometersPerHour)
    {
        
        var capacity = new VolumetricWeight(30);
        var location = new Location(59, 30);

        
        Exception? exception = Record.Exception(
            () => new Truck(capacity, new Speed(kilometersPerHour), location));

        
        Assert.IsType<ArgumentOutOfRangeException>(exception);
    }

    [Fact]
    public void Constructor_ZeroCapacity_ThrowsOutOfRange()
    {
        
        var zeroCapacity = new VolumetricWeight(0);

        
        Exception? exception = Record.Exception(
            () => new Truck(zeroCapacity, new Speed(60), new Location(59, 30)));

        
        Assert.IsType<ArgumentOutOfRangeException>(exception);
    }

    [Fact]
    public void Load_GoodsWithinCapacity_UpdatesCargo()
    {
        
        var lineItems = new List<LineItem> { new LineItem(_crate, 4), new LineItem(_barrel, 2) };
        var goods = new Goods(lineItems);

        
        LoadResult result = _truck.Load(goods);

        
        Assert.IsType<LoadResult.Success>(result);
        Assert.Equal(4, _truck.Cargo.CountUnits(_crate));
        Assert.Equal(2, _truck.Cargo.CountUnits(_barrel));
        Assert.Equal(new VolumetricWeight(22), _truck.Cargo.VolumetricWeight);
    }

    [Fact]
    public void Load_GoodsExactlyFillingCapacity_UpdatesCargo()
    {
        
        var crates = new Goods(new List<LineItem> { new LineItem(_crate, 10) });

        
        LoadResult result = _truck.Load(crates);

        
        Assert.IsType<LoadResult.Success>(result);
        Assert.Equal(_truck.Capacity, _truck.Cargo.VolumetricWeight);
    }

    [Fact]
    public void Load_GoodsOverCapacityIntoEmptyTruck_ReturnsFailureAndLeavesTruckEmpty()
    {
        
        Goods crates = CreateCrates(11);

        
        LoadResult result = _truck.Load(crates);

        
        Assert.IsType<LoadResult.Failure>(result);
        Assert.Empty(_truck.Cargo.LineItems);
    }

    [Fact]
    public void Load_GoodsOverFreeCapacity_ReturnsFailureAndKeepsCargo()
    {
        
        _truck.Load(CreateCrates(6));
        var barrels = new Goods(new List<LineItem> { new LineItem(_barrel, 3) });

        
        LoadResult result = _truck.Load(barrels);

        
        Assert.IsType<LoadResult.Failure>(result);
        Assert.Equal(6, _truck.Cargo.CountUnits(_crate));
        Assert.Equal(0, _truck.Cargo.CountUnits(_barrel));
    }

    [Fact]
    public void Unload_GoodsInCargo_DecreasesCargo()
    {
        
        _truck.Load(CreateCrates(6));
        Goods crates = CreateCrates(4);

        
        UnloadResult result = _truck.Unload(crates);

        
        Assert.IsType<UnloadResult.Success>(result);
        Assert.Equal(2, _truck.Cargo.CountUnits(_crate));
    }

    [Fact]
    public void Unload_MoreThanLoaded_ReturnsFailureAndKeepsCargo()
    {
        
        _truck.Load(CreateCrates(6));
        Goods crates = CreateCrates(7);

        
        UnloadResult result = _truck.Unload(crates);

        
        Assert.IsType<UnloadResult.Failure>(result);
        Assert.Equal(6, _truck.Cargo.CountUnits(_crate));
    }

    [Fact]
    public void DriveTo_OneDegreeNorth_ChangesLocationAndTakesTimeBySpeed()
    {
        
        var destination = new Location(60, 30);

        
        TimeSpan travelTime = _truck.DriveTo(destination);

        
        Assert.Equal(destination, _truck.Location);
        Assert.Equal(111.19, travelTime.TotalMinutes, 2);
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```

**19. `tests/Lab1.Tests/Warehousing/CrewTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Warehousing;

public class CrewTests
{
    private readonly Sku _crate;
    private readonly Crew _oneEmployeeCrew;

    public CrewTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));

        var employee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        _oneEmployeeCrew = new Crew(new List<Employee> { employee });
    }

    [Fact]
    public void Constructor_NoEmployees_ThrowsArgumentException()
    {
        
        var noEmployees = new List<Employee>();

        
        Exception? exception = Record.Exception(() => new Crew(noEmployees));

        
        Assert.IsType<ArgumentException>(exception);
    }

    [Fact]
    public void Plan_GoodsWithinMaxLoadOfOneEmployee_TakesOneRound()
    {
        
        Goods goods = CreateCrates(4);

        
        PlanningResult result = _oneEmployeeCrew.Plan(goods);

        
        PlanningResult.Success plan = Assert.IsType<PlanningResult.Success>(result);
        Assert.Equal(1, plan.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(6), plan.DispatchTime);
        Assert.Equal(TimeSpan.FromMinutes(2), plan.ReceivingTime);
    }

    [Fact]
    public void Plan_GoodsOverMaxLoadOfOneEmployee_TakesSeveralRounds()
    {
        
        Goods goods = CreateCrates(7);

        
        PlanningResult result = _oneEmployeeCrew.Plan(goods);

        
        PlanningResult.Success plan = Assert.IsType<PlanningResult.Success>(result);
        Assert.Equal(2, plan.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(12), plan.DispatchTime);
        Assert.Equal(TimeSpan.FromMinutes(8), plan.ReceivingTime);
    }

    [Fact]
    public void Plan_SkuOnlyStrongEmployeeCanLift_StrongEmployeeCarriesItAlone()
    {
        
        var quickEmployee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        var strongEmployee = new Employee(
            new VolumetricWeight(18),
            TimeSpan.FromMinutes(3),
            TimeSpan.FromMinutes(7));

        var crew = new Crew(new List<Employee> { quickEmployee, strongEmployee });
        var engine = new Sku(new SkuId(104), "Engine", new VolumetricWeight(15));
        var goods = new Goods(new List<LineItem> { new LineItem(engine, 2) });

        
        PlanningResult result = crew.Plan(goods);

        
        PlanningResult.Success plan = Assert.IsType<PlanningResult.Success>(result);
        Assert.Equal(2, plan.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(20), plan.DispatchTime);
    }

    [Fact]
    public void Plan_OneSkuHeavierThanEveryEmployee_ReturnsFailure()
    {
        
        var piano = new Sku(new SkuId(103), "Piano", new VolumetricWeight(20));
        var lineItems = new List<LineItem> { new LineItem(_crate, 2), new LineItem(piano, 1) };
        var goods = new Goods(lineItems);

        
        PlanningResult result = _oneEmployeeCrew.Plan(goods);

        
        Assert.IsType<PlanningResult.Failure>(result);
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```

**20. `tests/Lab1.Tests/Warehousing/WarehouseHandlingTimeTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Warehousing;

public class WarehouseHandlingTimeTests
{
    private readonly Employee _quickEmployee;
    private readonly Employee _strongEmployee;

    public WarehouseHandlingTimeTests()
    {
        _quickEmployee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        _strongEmployee = new Employee(
            new VolumetricWeight(18),
            TimeSpan.FromMinutes(3),
            TimeSpan.FromMinutes(7));
    }

    [Fact]
    public void Receive_OneEmployee_SkipsCarryingInLastRound()
    {
        
        var employees = new List<Employee> { _quickEmployee };
        Warehouse warehouse = CreateWarehouse(employees, new Goods());
        Goods goods = CreateCrates(8);

        
        ReceivingResult result = warehouse.Receive(goods);

        
        ReceivingResult.Success success = Assert.IsType<ReceivingResult.Success>(result);
        Assert.Equal(2, success.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(8), success.Duration);
    }

    [Fact]
    public void Dispatch_OneEmployee_CountsEveryRoundFully()
    {
        
        Goods stock = CreateCrates(8);
        var employees = new List<Employee> { _quickEmployee };
        Warehouse warehouse = CreateWarehouse(employees, stock);

        
        DispatchResult result = warehouse.Dispatch(stock);

        
        DispatchResult.Success success = Assert.IsType<DispatchResult.Success>(result);
        Assert.Equal(2, success.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(12), success.Duration);
    }

    [Fact]
    public void Receive_TwoEmployees_WaitsForSlowestExceptLastCarrying()
    {
        
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, new Goods());
        Goods goods = CreateCrates(15);

        
        ReceivingResult result = warehouse.Receive(goods);

        
        ReceivingResult.Success success = Assert.IsType<ReceivingResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(13), success.Duration);
    }

    [Fact]
    public void Dispatch_TwoEmployees_WaitsForSlowestInEveryRound()
    {
        
        Goods stock = CreateCrates(15);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, stock);

        
        DispatchResult result = warehouse.Dispatch(stock);

        
        DispatchResult.Success success = Assert.IsType<DispatchResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(20), success.Duration);
    }

    [Fact]
    public void Dispatch_GoodsFitFirstEmployee_OnlyHisTimeCounts()
    {
        
        Goods stock = CreateCrates(3);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, stock);

        
        DispatchResult result = warehouse.Dispatch(stock);

        
        DispatchResult.Success success = Assert.IsType<DispatchResult.Success>(result);
        Assert.Equal(TimeSpan.FromMinutes(6), success.Duration);
    }

    [Fact]
    public void Dispatch_GoodsOverWholeCrewLoad_CountsRounds()
    {
        
        Goods stock = CreateCrates(25);
        var employees = new List<Employee> { _quickEmployee, _strongEmployee };
        Warehouse warehouse = CreateWarehouse(employees, stock);

        
        DispatchResult result = warehouse.Dispatch(stock);

        
        DispatchResult.Success success = Assert.IsType<DispatchResult.Success>(result);
        Assert.Equal(3, success.RoundsCount);
        Assert.Equal(TimeSpan.FromMinutes(30), success.Duration);
    }

    private Warehouse CreateWarehouse(IReadOnlyCollection<Employee> employees, Goods stock)
    {
        return new Warehouse(
            new WarehouseId(7),
            new Location(59, 30),
            new VolumetricWeight(90),
            new Crew(employees),
            stock);
    }

    private Goods CreateCrates(int count)
    {
        var crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));
        var lineItems = new List<LineItem> { new LineItem(crate, count) };

        return new Goods(lineItems);
    }
}
```

**21. `tests/Lab1.Tests/Warehousing/WarehouseStockTests.cs`** — новое содержимое целиком:

```csharp
using Itmo.ObjectOrientedProgramming.Lab1.Inventory;
using Itmo.ObjectOrientedProgramming.Lab1.Navigation;
using Itmo.ObjectOrientedProgramming.Lab1.Warehousing;
using Xunit;

namespace Itmo.ObjectOrientedProgramming.Lab1.Tests.Warehousing;

public class WarehouseStockTests
{
    private readonly Sku _crate;
    private readonly Crew _crew;
    private readonly Warehouse _warehouse;

    public WarehouseStockTests()
    {
        _crate = new Sku(new SkuId(101), "Crate", new VolumetricWeight(3));

        var quickEmployee = new Employee(
            new VolumetricWeight(12),
            TimeSpan.FromMinutes(2),
            TimeSpan.FromMinutes(4));

        var strongEmployee = new Employee(
            new VolumetricWeight(18),
            TimeSpan.FromMinutes(3),
            TimeSpan.FromMinutes(7));

        _crew = new Crew(new List<Employee> { quickEmployee, strongEmployee });

        _warehouse = new Warehouse(
            new WarehouseId(1),
            new Location(59, 30),
            new VolumetricWeight(60),
            _crew,
            CreateCrates(5));
    }

    [Fact]
    public void Constructor_ValidAttributes_KeepsAttributes()
    {
        
        var id = new WarehouseId(2);
        var location = new Location(59, 30);
        var capacity = new VolumetricWeight(60);
        Goods stock = CreateCrates(5);

        
        var warehouse = new Warehouse(id, location, capacity, _crew, stock);

        
        Assert.Equal(id, warehouse.Id);
        Assert.Equal(location, warehouse.Location);
        Assert.Equal(capacity, warehouse.Capacity);
        Assert.Equal(5, warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Constructor_StockOverCapacity_ThrowsArgumentException()
    {
        
        Goods tooManyCrates = CreateCrates(21);

        
        Exception? exception = Record.Exception(() => new Warehouse(
            new WarehouseId(2),
            new Location(59, 30),
            new VolumetricWeight(60),
            _crew,
            tooManyCrates));

        
        Assert.IsType<ArgumentException>(exception);
    }

    [Fact]
    public void Receive_GoodsWithinFreeCapacity_IncreasesStock()
    {
        
        Goods goods = CreateCrates(10);

        
        ReceivingResult result = _warehouse.Receive(goods);

        
        Assert.IsType<ReceivingResult.Success>(result);
        Assert.Equal(15, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Receive_GoodsOverFreeCapacity_ReturnsFailureAndKeepsStock()
    {
        
        Goods goods = CreateCrates(16);

        
        ReceivingResult result = _warehouse.Receive(goods);

        
        Assert.IsType<ReceivingResult.Failure>(result);
        Assert.Equal(5, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Receive_GoodsTooHeavyForEveryEmployee_ReturnsFailureAndKeepsStock()
    {
        
        var piano = new Sku(new SkuId(103), "Piano", new VolumetricWeight(20));
        var lineItems = new List<LineItem> { new LineItem(piano, 1) };
        var goods = new Goods(lineItems);

        
        ReceivingResult result = _warehouse.Receive(goods);

        
        Assert.IsType<ReceivingResult.Failure>(result);
        Assert.Equal(0, _warehouse.Stock.CountUnits(piano));
        Assert.Equal(5, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Receive_EmptyGoods_ThrowsArgumentException()
    {
        
        var emptyGoods = new Goods();

        
        Exception? exception = Record.Exception(() => _warehouse.Receive(emptyGoods));

        
        Assert.IsType<ArgumentException>(exception);
        Assert.Equal(5, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Dispatch_GoodsInStock_DecreasesStock()
    {
        
        Goods goods = CreateCrates(3);

        
        DispatchResult result = _warehouse.Dispatch(goods);

        
        Assert.IsType<DispatchResult.Success>(result);
        Assert.Equal(2, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Dispatch_MoreThanInStock_ReturnsFailureAndKeepsStock()
    {
        
        Goods goods = CreateCrates(6);

        
        DispatchResult result = _warehouse.Dispatch(goods);

        
        Assert.IsType<DispatchResult.Failure>(result);
        Assert.Equal(5, _warehouse.Stock.CountUnits(_crate));
    }

    [Fact]
    public void Dispatch_GoodsTooHeavyForEveryEmployee_ReturnsFailureAndKeepsStock()
    {
        
        var piano = new Sku(new SkuId(103), "Piano", new VolumetricWeight(20));
        var pianos = new Goods(new List<LineItem> { new LineItem(piano, 1) });

        var warehouse = new Warehouse(
            new WarehouseId(3),
            new Location(59, 30),
            new VolumetricWeight(60),
            _crew,
            pianos);

        
        DispatchResult result = warehouse.Dispatch(pianos);

        
        Assert.IsType<DispatchResult.Failure>(result);
        Assert.Equal(1, warehouse.Stock.CountUnits(piano));
    }

    private Goods CreateCrates(int count)
    {
        var lineItems = new List<LineItem> { new LineItem(_crate, count) };

        return new Goods(lineItems);
    }
}
```
