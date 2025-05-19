# Anti-Crash-Dumper7
Many changes over years of use to prevent null pointers from killing the sdk. Some games have garbage collectors or other things that break vanilla dumper7.

# personal notes of changes:


## Check for a specific index that always crashes the dumper. IDK what it actually represents in game.


`//DependencyManager.cpp`
`void DependencyManager::VisitIndexAndDependencies(int32 Index, OnVisitCallbackType Callback) const`
`{`
    `auto& [IterationHitCounter, Dependencies] = AllDependencies.at(Index);`

    `if (IterationHitCounter >= CurrentIterationHitCount)`
        `return;`

    `IterationHitCounter = CurrentIterationHitCount;`

    `for (int32 Dependency : Dependencies)`
    `{`
        `VisitIndexAndDependencies(Dependency, Callback);`
    `}`

    `if (Index == 0x0001291d) //idk what this even is but I always crash on it`
        `return;`

    `Callback(Index);`
`}`

## Checker Outer Object pointer in Struct loop

`//CppGenerator.cpp`
`void CppGenerator::GenerateNameCollisionsInl(StreamType& NameCollisionsFile)`
`{`
    `namespace CppSettings = Settings::CppGenerator;`

    `WriteFileHead(NameCollisionsFile, nullptr, EFileType::NameCollisionsInl, "FORWARD DECLARATIONS");`

    `const StructManager::OverrideMaptType& StructInfoMap = StructManager::GetStructInfos();`
    `const EnumManager::OverrideMaptType& EnumInfoMap = EnumManager::GetEnumInfos();`

    `std::unordered_map<int32 /* PackageIdx */, std::pair<std::string, int32>> PackagesAndForwardDeclarations;`

    `for (const auto& [Index, Info] : StructInfoMap)`
    `{`
        `if (StructManager::IsStructNameUnique(Info.Name))`
            `continue;`

        `UEStruct Struct = ObjectArray::GetByIndex<UEStruct>(Index);`

        `if (!Struct.UEField::GetOuter()) //added this`
            `continue;`

## void ObjectArray::DumpObjects function fails on null object in loop

I attempt to fix it by simply skipping null objects and skipping objects who don't return valid pointer address.

`void ObjectArray::DumpObjects(const fs::path& Path, bool bWithPathname)
{
	std::ofstream DumpStream(Path / "GObjects-Dump.txt");

	DumpStream << "Object dump by Dumper-7\n\n";
	DumpStream << (!Settings::Generator::GameVersion.empty() && !Settings::Generator::GameName.empty() ? (Settings::Generator::GameVersion + '-' + Settings::Generator::GameName) + "\n\n" : "");
	DumpStream << "Count: " << Num() << "\n\n\n";

	for (auto Object : ObjectArray())
	{
		//mark's code
		if (!Object) 
			continue;
		else if (!Object.GetAddress())
			continue;
		//end mark's code
		else if (!bWithPathname) //mark added "else" to the if so this block flows faster
		{
			DumpStream << std::format("[{:08X}] {{{}}} {}\n", Object.GetIndex(), Object.GetAddress(), Object.GetFullName());
		}
		else
		{
			DumpStream << std::format("[{:08X}] {{{}}} {}\n", Object.GetIndex(), Object.GetAddress(), Object.GetPathName());
		}
	}

	DumpStream.close();
}`

## UEObject UEObject::GetOuter() crashes because outer is NULL

Object was NULL, the flaw from the patch above to the object dumper? Or it's a new loop through the objects for the creation of something else. To fix this I will perform another patch.

`UEObject UEObject::GetOuter() const
{
	if(Off::UObject::Outer && Object) //mark's code to check valid pointers & object
		return UEObject(*reinterpret_cast<void**>(Object + Off::UObject::Outer));
	else { return UEObject(nullptr); } //marks' code to return fake object
}`

This patch has a direct impact on **Checker Outer Object pointer in Struct loop** but luckily I've pre-patched that to ignore fake returns on GetOuter().

## UEClass UEObject::GetClass() has NULL Object, shit runs downhill

Left over problem from the above issues. It's just a trickle down effect. Eventually I'll get it all fixed.

![[Pasted image 20250212012640.png]]

![[Pasted image 20250212012646.png]]

I have added extra checks for null pointer within Object before attempting to use it.
✔️✔️✔️✔️✔️✔️✔️✔️ Fixed it with the code bellow.

`void ObjectArray::DumpObjects(const fs::path& Path, bool bWithPathname)
{
	std::ofstream DumpStream(Path / "GObjects-Dump.txt");

	DumpStream << "Object dump by Dumper-7\n\n";
	DumpStream << (!Settings::Generator::GameVersion.empty() && !Settings::Generator::GameName.empty() ? (Settings::Generator::GameVersion + '-' + Settings::Generator::GameName) + "\n\n" : "");
	DumpStream << "Count: " << Num() << "\n\n\n";

	for (auto Object : ObjectArray())
	{
		//mark's code
		if (&Object == nullptr) 
			continue;
		void* addr = Object.GetAddress();
		if (addr == nullptr || (uint8_t*)addr == (uint8_t*)0)
			continue;
		//end mark's code
		else if (!bWithPathname) //mark added "else" to the if so this block flows faster
		{
			DumpStream << std::format("[{:08X}] {{{}}} {}\n", Object.GetIndex(), addr, Object.GetFullName());
		}
		else
		{
			DumpStream << std::format("[{:08X}] {{{}}} {}\n", Object.GetIndex(), addr, Object.GetPathName());
		}
	}

	DumpStream.close();
}`


## 2/13/2025 small update to game, causes null objects in the GenerateStruct function

DSGen::ClassHolder DumpspaceGenerator::GenerateStruct(const StructWrapper& Struct)
added the code bellow to help???
`if (!&Struct || !Struct.IsValid())`
	`return StructOrClass;`

That appears to have made a valid dump. I'll have to be careful going forward to make sure the dump is correct on the parts I use. I also added a return of struct prefix "nigger" somwhere.
