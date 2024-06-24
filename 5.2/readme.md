# Unreal Engine 5.2

Follow this guide sequentially.

## 1. Configure your Visual Studio

1. Download Visual Studio 2022 - Version 17.9.6. 
    
    *Other versions may work, but this is the version this guide is tested with.*

1. Go to the Visual Studio Installer application. 
    1. Install the following modules:
        1. "IDE Support for Unreal Engine"
        1. "Unreal Engine Test Adapter"

# 2. Create a folder for your "Fixtures"

"Fixture" is a word used to explain the idea of a class that provides a defined, reliable and consistent context for your tests. We need this as some functionality requires actors to be present within a UE World context for them to work.

1. Create the following file in this folder structure:

    `{ProjectName}/Source/{ProjectName}/Public/Tests/Fixtures/WorldFixture.h`

1. Add the following code to the file replacing {ProjectName} accordingly:

	```cpp
	#pragma once

    #include "CoreMinimal.h"
    #include "EngineUtils.h"

    /**
     * Thanks to: https://minifloppy.it/posts/2024/automated-testing-specs-ue5/
     */
    class {ProjectName}_API FWorldFixture
    {
    public:

        explicit FWorldFixture(const FURL& URL = FURL())
        {
            if (GEngine != nullptr)
            {
                static uint32 WorldCounter = 0;
                const FString WorldName = FString::Printf(TEXT("WorldFixture_%d"), WorldCounter++);

                if (UWorld* World = UWorld::CreateWorld(EWorldType::Game, false, *WorldName, GetTransientPackage()))
                {
                    FWorldContext& WorldContext = GEngine->CreateNewWorldContext(EWorldType::Game);
                    WorldContext.SetCurrentWorld(World);

                    World->InitializeActorsForPlay(URL);
                    if (IsValid(World->GetWorldSettings()))
                    {
                        // Need to do this manually since world doesn't have a game mode
                        World->GetWorldSettings()->NotifyBeginPlay();
                        World->GetWorldSettings()->NotifyMatchStarted();
                    }
                    World->BeginPlay();

                    WeakWorld = MakeWeakObjectPtr(World);
                }
            }
        }

        UWorld* GetWorld() const { return WeakWorld.Get(); }

        ~FWorldFixture()
        {
            UWorld* World = WeakWorld.Get();
            if (World != nullptr && GEngine != nullptr)
            {
                World->BeginTearingDown();

                // Make sure to cleanup all actors immediately
                // DestroyWorld doesn't do this and instead waits for GC to clear everything up
                for (auto It = TActorIterator<AActor>(World); It; ++It)
                {
                    It->Destroy();
                }

                GEngine->DestroyWorldContext(World);
                World->DestroyWorld(false);
            }
        }

    private:

        TWeakObjectPtr<UWorld> WeakWorld;
    };
	```

# 3. Create a folder for your Tests

First - A cautionary tale: Creating a file that holds all of the tests for a certain class leads to monolithic files. Therefore:

1. Create the following file in this folder structure whereby `Actors` defines a grouping for actors, `BeerActor` is the name of the class, `DrinkBeer` is the name of the function being tested and the `BeerActor_` part prevents namespace issues:

    `{ProjectName}/Source/{ProjectName}/Private/Tests/Actors/BeerActor/BeerActor_DrinkBeer.spec.cpp`

# 4. Writing your first test

## The implentation

Let's imagine you have a class called `BeerActor` and a function called `DrinkBeer` that you want to test. It might look like this:

`{ProjectName}/Source/{ProjectName}/Public/Actors/BeerActor.h`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "BeerActor.generated.h"

UCLASS()
class {ProjectName}_API ABeerActor : public AActor
{
	GENERATED_BODY()
	
public:	
	// Sets default values for this actor's properties
	ABeerActor();

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	UPROPERTY(BlueprintReadWrite, VisibleDefaultsOnly, Category = "Custom")
	float DecreaseTemperatureBy;

	UFUNCTION(BlueprintNativeEvent, Category = "Custom")
	void DrinkBeer();
};
```

`{ProjectName}/Source/{ProjectName}/Private/Actors/BeerActor.cpp`

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "Actors/BeerActor.h"
#include "Controllers/MainPlayerController.h"
#include "Characters/MainCharacter.h"

// Sets default values
ABeerActor::ABeerActor()
{
	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

}

// Called when the game starts or when spawned
void ABeerActor::BeginPlay()
{
	Super::BeginPlay();

}

// Called every frame
void ABeerActor::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

}

void ABeerActor::DrinkBeer_Implementation()
{
	UWorld* World = GetWorld();

	if (World) {
		AMainPlayerController* Controller = World->GetFirstPlayerController<AMainPlayerController>();

		if (Controller)
		{
			AMainCharacter* Character = Cast<AMainCharacter>(Controller->GetCharacter());

			if (Character)
			{
				Character->DecreaseTemperature(DecreaseTemperatureBy);
			}
		}
	}
}
```

## The test

`{ProjectName}/Source/{ProjectName}/Private/Tests/Actors/BeerActor/BeerActor_DrinkBeer.spec.cpp`

```cpp
#include "CoreMinimal.h"
#include "Actors/BeerActor.h"
#include "Characters/MainCharacter.h"
#include "Controllers/MainPlayerController.h"
#include "PlayerStates/MainPlayerState.h"
#include "Misc/AutomationTest.h"
#include "Tests/Fixtures/WorldFixture.h"

// BeerActorDrinkBeerSpec: A name for the class. It's used for the Define method
// "{ProjectName}.Actors.BeerActor.DrinkBeerSpec": A name for the spec. Dots "." are used in Unreal Engine Editor to organise tests.
// EAutomationTestFlags::ApplicationContextMask: This flag does something... <shrugs>
// EAutomationTestFlags::ProductFilter: This flag is used to categorise your test in the Unreal Engine Editor.
BEGIN_DEFINE_SPEC(BeerActorDrinkBeerSpec, "{ProjectName}.Actors.BeerActor.DrinkBeerSpec", EAutomationTestFlags::ApplicationContextMask | EAutomationTestFlags::ProductFilter)

// Within this block, you can define variables that you use across your "It" tests
TUniquePtr<FWorldFixture> WorldFixture;
ABeerActor* Actor;
AMainPlayerController* Controller;
AMainCharacter* Character;
AMainPlayerState* PlayerState;
// ---

END_DEFINE_SPEC(BeerActorDrinkBeerSpec)

void BeerActorDrinkBeerSpec::Define()
{
	Describe("DrinkBeer", [this]()
	{
		BeforeEach([this]()
		{
			// Make use of our fixture
			WorldFixture = MakeUnique<FWorldFixture>();

			// Spawn our beer actor
			Actor = WorldFixture->GetWorld()->SpawnActor<ABeerActor>();

			// Spawn other stuff needed for this example. It might be different for you!
			Controller = WorldFixture->GetWorld()->SpawnActor<AMainPlayerController>();
			Character = WorldFixture->GetWorld()->SpawnActor<AMainCharacter>();
			PlayerState = NewObject<AMainPlayerState>();
			
			// Set up the player state, Controller, character, etc.
			WorldFixture->GetWorld()->AddController(Controller);
			Character->SetPlayerState(PlayerState);
			Controller->Possess(Character);
		});

		It("should decrease the character temperature", [this]()
		{
			// Arrange
			Actor->DecreaseTemperatureBy = 0.2;
			PlayerState->Temperature = 1;

			// Act
			Actor->DrinkBeer();

			// Assert
			TestEqual("Temperature value decreased as expected", PlayerState->Temperature, 0.8f);
		});
	});
}
```

You can see here the `DrinkBeer` Describe block groups tests (`It`) together. In this case, for this single method under the `BeerActor` class.

The `BeforeEach` runs before every `It` method is called. This is where we set up the world, the actors, player states, etc.

The `It` respresents each test you want to perform on that method. Inside that `It` lamdba (`[](){...}`), you can define an `// Arrange` section.

### Arrange

Be very specific. What are the explicit requirements of your test? Make it obvious. The clearer it is, the less confusing it will be when returning back to it. If something doesn't need specific wording, use the string `test`. If something doesn't need a specific number, use something obvious like `123`, etc. Be as clear as possible and as directly related to the unit test as possible.

### Act

Now it's time to Act with the `// Act` section. This is where you call the method you want to test, with all the parts you set up in the `// Arrange` section. In the above example, we call `Actor->DrinkBeer()`

### Assert

Here, you can use the [following methods](https://docs.unrealengine.com/4.26/en-US/API/Runtime/Core/Misc/FAutomationTestBase/) to make assertions about the results:

- TestEqual
- TestEqualInsensitive
- TestFalse
- TestInvalid
- TestNotEqual
- TestNotNull
- TestNotSame
- TestNull
- TestSame
- TestTrue
- TestValid

In this case, we are using `TestEqual` to check if the temperature has decreased by the expected amount.

# 5. Running your tests

## Running tests in Unreal Engine

1. Compile and run your project so that Unreal Engine Editor is running.
1. Visit `Tools/Session Frontend` in the Unreal Engine Editor.
1. Click `Automation`
1. Click on the dropdown that says `Standard Tests` and choose `Product Tests`. (This was defined by this `EAutomationTestFlags::ProductFilter` flag). 
1. Find your test and select the checkbox to mark for running.
1. Click "Start Tests"
1. You can debug at this point by returning to your Visual Studio editor and setting breakpoints inside your test. Just note that optimizations may need to be turned off.

## Running tests using command line (CLI)

1. Open a command prompt and type this in, modifying the command tokens to match your requirements. For example, `{Username}`, `{ProjectName}` and `{TestName}`. `{TestName}` can be used as a sort of search, for the tests you want to run.

    ```cmd
    "C:\Program Files\Epic Games\UE_5.2\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "C:\Users\{Username}\Unreal Projects\{ProjectName}\{ProjectName}.uproject" -execcmds="Automation RunTests {TestName};Quit" -stdout -unattended -NOSPLASH -NullRHI
    ```

1. Now hit run, and you should see the tests run in the command prompt.
1. If you want to see colours for errors, try using PowerShell instead and running the following:

    ```powershell
    $command = "& 'C:\Program Files\Epic Games\UE_5.2\Engine\Binaries\Win64\UnrealEditor-Cmd.exe' 'C:\Users\{Username}\Unreal Projects\{ProjectName}\{ProjectName}.uproject' -execcmds='Automation RunTests {TestName};Quit' -stdout -unattended -NOSPLASH -NullRHI"
    Invoke-Expression $command | ForEach-Object {
        if ($_ -match "Error") {
            Write-Host $_ -ForegroundColor Red
        } elseif ($_ -match "Success") {
            Write-Host $_ -ForegroundColor Green
        } else {
            Write-Host $_ -ForegroundColor White
        }
    }
    ```

For what it's worth, here's an explanation of the command.

- `execcmds="Automation RunTests Loyly;Quit"`: This tells the Unreal Engine command-line editor to run automation tests that match the "Loyly" filter, and then quit the editor once the tests have been completed. It does not require an additional specification for EAutomationTestFlags::EditorContext because the test itself defines the context (and other flags) necessary for execution.
- `stdout`: Forces the engine to output logs to the standard output, which can be very useful for CI systems to capture and analyze the output directly.
- `unattended`: Runs the engine in a mode that does not require user interaction, suitable for automated tasks like continuous integration.
- `NOSPLASH`: Disables the splash screen on startup, which is generally preferred for automated tasks to reduce overhead and potential graphical issues.
- `NullRHI`: This is a crucial flag for running tests in environments without a dedicated GPU or where you do not want to initialize the full rendering hardware interface. It makes the tests run with a null renderer, which can prevent rendering-related operations but allows most other engine functionality to be tested.

### Running tests using Visual Studio

1. Visit `Test/Test Explorer` in Visual Studio.
1. Note that, as long as you configured your Visual Studio correctly, you should see your tests here.
1. You can filter using the search bar. There is a lot of built in tests here so it may be useful to filter by your project name.

# References

1. [Unreal Engine and C++ Game Development Made Easy with Visual Studio 2022](https://devblogs.microsoft.com/cppblog/unreal-engine-cpp-game-development-made-easy-visual-studio-2022/)
    
    *This blog makes mention of the Visual Studio tooling that allows for Unit Testing.*

1. [Automated Testing Specs UE5](https://minifloppy.it/posts/2024/automated-testing-specs-ue5/)

1. [Automation Spec In Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/automation-spec-in-unreal-engine)