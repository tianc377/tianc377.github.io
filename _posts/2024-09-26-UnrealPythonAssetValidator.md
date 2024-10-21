---
title:  "Unreal Python Asset Validator"
date:   2024-09-26 18:00
categories: [unrealtools]
tags: [Python]
image: /post-img/unrealtools/ue-validator-cover.png
published: true
---


Recently, I’ve been exploring the implementation of validation in Unreal Engine. Validation is a crucial step in production, helping to minimize the time spent fixing assets after submission. By enhancing the validation process, we can reduce errors and decrease the time wasted on communication and bug fixing. Different applications accommodate various approaches to validation. In the case of Unreal Engine, it has built-in source control tools implemented, it supports custom validation by using its API. 

## unreal.EditorValidatorBase
In Unreal, when you right-click an asset, there’s an option in the popup menu called `Assets Acitons` → `Validate Asset`. This button activates the validators currently available in the engine. However, what I want to achieve is to run validation on the files in the selected changelist when submitting.

I started exploring Unreal’s Python API, and came across something that looks promising: the `unreal.EditorValidatorBase` class. Within this class, there are several functions that seem suitable for validation, such as `validate_loaded_asset`. 
I also found a very helpful page on the Unreal Community Wiki about the **[Asset Validator](https://unrealcommunity.wiki/asset-validator-python-pddba19x)**. This page provides a brief overview of how to implement the `unreal.EditorValidatorBase` class. 
Essentially, you need to create a concrete subclass of `unreal.EditorValidatorBase`, override its functions, and then register your new validator with Unreal’s `EditorValidatorSubsystem` to activate your rules. You can set up multiple rules to accommodate different types of assets.

<br />

### k2_validate_loaded_asset

 ```python
@unreal.uclass()
class CTTextureValidator(unreal.EditorValidatorBase):
    def __init__(self, *arg, **kwds):
        super().__init__(*arg, **kwds)

    @unreal.ufunction(override=True)
    def k2_validate_loaded_asset(self, asset):
        flag = unreal.DataValidationResult.VALID
        prefix = 'T_'
        if asset.get_name().startswith(prefix):
            unreal.EditorValidatorBase.asset_passes(self, asset) 
        else:
            unreal.EditorValidatorBase.asset_fails(self, asset, f"【 Asset {asset.get_name()} has incorrect prefix, it must start with [{prefix}] 】")
            fix_prefix(prefix, asset)
            flag = unreal.DataValidationResult.INVALID
        if (asset.blueprint_get_size_x()>=4096) or (asset.blueprint_get_size_y()>=4096):
            flag = unreal.DataValidationResult.INVALID
            unreal.EditorValidatorBase.asset_fails(self, asset, f"【 Asset {asset.get_name()} is too large (>2048), please scale it before submit】")
        else:
            unreal.EditorValidatorBase.asset_passes(self, asset)
        return flag

 ```
In this block of code, I’ve implemented a texture validation function. Using `k2_validate_loaded_asset`, I verify whether the asset has the correct prefix "T_" and check if the texture’s width or height is greater than or equal to 4096, as sizes of that dimension are not acceptable, then return a `DataValidationResult`. 

I used two functions of `unreal.EditorValidatorBase` class: `asset_passes()` and `asset_fails()`. Additionally, the Unreal Python API also has an `asset_warning()` function. The latter two functions allow you to insert log messages, while `asset_fails()` will raise an error message and block submission:

![asset-fails](/post-img/unrealtools/ue-validator/asset-fails.png){: width="80%" .shadow} <br />

<br />

### k2_can_validate_asset

As I mentioned, you can set up multiple rules to accommodate different types of assets. But how does each rule know which assets to target? In this case, `k2_can_validate_asset` is very useful:

![can-validate-asset](/post-img/unrealtools/ue-validator/can-validate-asset.png){: width="80%" .shadow} 

you can define it to return True or False based on the asset being validated. Like here in my script, I filtered to let it only work for `Texture2D` type of files:

 ```python
 ...
    @unreal.ufunction(override=True)
    def k2_can_validate_asset(self,asset):
        print("--------Texture Validation Executed-------")
        if asset.get_class().get_class_path_name().asset_name == 'Texture2D':
            return True  
        else:
            return False
```


<br />

### DataValidationUsecase

Another similar function is `k2_can_validate`, which determines whether the validator should be applied based on the use case:

![can-validate-usecase](/post-img/unrealtools/ue-validator/can-validate-usecase.png){: width="80%" .shadow} 

There are six [DataValidationUsecase](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/DataValidationUsecase?application_version=5.4#unreal.DataValidationUsecase): `COMMANDLET`, `MANUAL`, `NONE`, `PRE_SUBMIT`, `SAVE`, and `SCRIPT`. For example, if the use case is set to SAVE, the validator will be activated when you save assets.


<br />

## unreal.EditorValidatorSubsystem 

```python
if __name__ == "__main__":
    editor_validator_subsystem = unreal.get_editor_subsystem(unreal.EditorValidatorSubsystem)
    ct_texture_validator = CTTextureValidator()

    editor_validator_subsystem.validate_changelists([unreal.DataValidationChangelist()], unreal.ValidateAssetsSettings(validation_usecase = unreal.DataValidationUsecase.PRE_SUBMIT))

    editor_validator_subsystem.add_validator(ct_texture_validator)
```
After completing the class and function overrides, the final step is to register the new validator with [unreal.EditorValidatorSubsystem](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/EditorValidatorSubsystem?application_version=5.4). To do this, you'll need to call `get_editor_subsystem`, instantiate the new validator you created, and then use `add_validator` to append it.

### validate_changelists
I also utilized the `validate_changelists` function from the `unreal.EditorValidatorSubsystem` class. This function can validate assets in changelists and takes two parameters: `[DataValidationChangelist]` and `ValidateAssetsSettings`. Within `ValidateAssetsSettings`, there are several editor properties available for use:

![validate-assets-settings](/post-img/unrealtools/ue-validator/validate-assets-settings.png){: width="80%" .shadow} 

In my script, I configured it with the `PRE_SUBMIT` use case, ensuring that the changelist being submitted will be validated by my custom validator.




## Add Multiple Validators
Adding multiple validators is quite straightforward. Simply need to create the class and override the necessary functions, just as you did with the first validator. After that, register the new validator with the subsystem:

```python
if __name__ == "__main__":
    editor_validator_subsystem = unreal.get_editor_subsystem(unreal.EditorValidatorSubsystem)
    ct_texture_validator = CTTextureValidator()
    ct_material_validator = CTMaterialValidator()


    editor_validator_subsystem.validate_changelists([unreal.DataValidationChangelist()], unreal.ValidateAssetsSettings(validation_usecase = unreal.DataValidationUsecase.PRE_SUBMIT))

    editor_validator_subsystem.add_validator(ct_texture_validator)
    editor_validator_subsystem.add_validator(ct_material_validator)
```


## Add Auto Fix Function

For validations like identifying incorrect prefixes, it makes sense to provide an auto-fix renaming feature, especially in Unreal Engine. Renaming can often be confusing, as Unreal creates a redirector to redirect references from the old asset location to the new one.

### Redirectors

![redirectors](/post-img/unrealtools/ue-validator/redirector.png){: width="90%" .shadow} 

When you move or rename an asset in UE4, a redirector is left in its original location. This ensures that packages not currently loaded but referencing the asset can find it in its new location. Unreal also offers a utility to clean up these redirectors. You can right-click the redirector and select "Update Redirector References," or right-click the folder containing the redirector and choose the same option.

However, since redirectors are hidden by default, it's easy to overlook them. If you later attempt to rename an asset with the same name, you'll see a warning that an asset already exists and have no idea why it's saying that. 

So I added an auto fix function with update redirectors right after it:

```python
def fix_redirectors(asset_path):
    asset_data = unreal.EditorAssetLibrary.find_asset_data(asset_path)
    asset_class = str(asset_data.asset_class_path.asset_name)            #<Struct 'TopLevelAssetPath' (0x00000B2175F52C9C) {package_name: "/Script/CoreUObject", asset_name: "ObjectRedirector"}>
    if (asset_class == 'ObjectRedirector'):
        unreal.CTAdditionalFunctions.fixup_referencers([asset_path])

def fix_prefix(prefix, asset):
    validator_message = unreal.EditorDialog.show_message('Validation Error', f'{asset.get_name()} has an invalid prefix.' + 'Click YES for an auto-fix.', unreal.AppMsgType.YES_NO, unreal.AppReturnType.OK)
    if validator_message == 1:
        
        old_path = asset.get_path_name().split('.', 1)[0]
        print('----------------------old_path:' + old_path)
        base_path = old_path.replace(asset.get_name() + '.' + asset.get_name(), '')
        asset_new_name = prefix + asset.get_name().split('_', 1)[1]
        new_path = old_path.replace(asset.get_name(), asset_new_name) 
        print('----------------------new_path:' + new_path)
        # new_path = base_path + asset_new_name + '.' + asset_new_name
        unreal.EditorAssetLibrary.rename_asset(old_path, new_path)   
        assets_in_base_path = unreal.EditorAssetLibrary.list_assets(base_path, True, True)
        for asset_path in assets_in_base_path:
            fix_redirectors(asset_path.split('.', 1)[0])
    else:
        pass 

```

### Create Fix Redirectors function from C++ 
Well for the function `fix_prefix` there's nothing much to say. The tricky one is the `fix_redirectors` because there's no python API is for updating the redirectors references, but maybe I can find the C++ function of this `Update Redirector References` button? 

![redirectors](/post-img/unrealtools/ue-validator/fix-redirectors-ui.png){: width="90%" .shadow} 

So I searched the Label name of that menu by using ripgrep, and found:
![redirectors](/post-img/unrealtools/ue-validator/rg-find-redirector.png){: width="90%" .shadow} 


`AssetFolderContextMenu.cpp:`
![redirectors](/post-img/unrealtools/ue-validator/fix-redirectors-c.png){: width="100%" .shadow}

In this cpp file, the last few lines are the part that actually gather redirectors of objects and fix the directors:
```C++
    TArray<UObjectRedirector*> Redirectors;
    for (UObject* Object : Objects)
    {
        if (UObjectRedirector* Redirector = Cast<UObjectRedirector>(Object))
        {
            Redirectors.Add(Redirector);
        }
    }
// Load the asset tools module
    FAssetToolsModule& AssetToolsModule = FModuleManager::LoadModuleChecked<FAssetToolsModule>(TEXT("AssetTools"));
    AssetToolsModule.Get().FixupReferencers(Redirectors, true, ERedirectFixupMode::PromptForDeletingRedirectors);
```
What it needs is just a list of objects. 
Therefore the C++ function can take the path of the folder that needs clean up to be the parameter passed in:

![redirectors](/post-img/unrealtools/ue-validator/fix-redirectors-c-python.png){: width="90%" .shadow}

This additional C++ file is added under my Plugin's folder that I created in the last few posts, the function is also registered into the BlueprintFunctionLibrary, it can be both used in Blueprint ![bp](/post-img/unrealtools/ue-validator/in-bp.png){: .left .shadow} and with Python. 


To use this C++ function in python, just simply call it with:
```python
unreal.CTAdditionalFunctions.fixup_referencers([asset_path])
```

<br />

## Lastly

I didn’t add many functionalities to the validator, as my main goal was to understand the process of creating and utilizing it. Below gifs demonstrate how it works during submission and saving:

As you can see, when submitting a problematic asset, the changelist will show a red error line and greyout the submit button:

![texture-size-validator](/post-img/unrealtools/ue-validator/texture-size-validator.gif){: .left .shadow} 

It can also detect when trying to save an invalid asset, and activate the auto fix I added:
![texture-prefix-fix](/post-img/unrealtools/ue-validator/texture-prefix-fix.gif){: .left .shadow}<br />

..


<br />

## Unresolved Issues 

There's something that's been bothering me. When you right-click the small icon in the bottom-right corner, you're presented with options like `Submit Files` and `View Changelist`:
![revision-control-button](/post-img/unrealtools/ue-validator/revision-control-button.png){: .left .shadow}

The `Submit Files` button allows you to submit all checked-out or selected files with an automatically created changelist, but it can bypass validation. This feels like a strange design choice, and I haven't found any functions that can address the issue. Essentially, this button makes the validation system ineffective, as it doesn't prevent users from submitting invalid assets.

So, I dug into the original C++ files. In `SSourceControlChangelists.cpp`, there's a function called `GetChangelistValidationResult`, which handles validation properly by adding relevant UI behavior—like displaying errors and greying out the submit button if validation fails. However, there's no similar process in `SSourceControlSubmit.cpp`, which is where the `Submit Content` button's functionality resides. I really don't understand the reason of this design. 

That led me to think of three potential workarounds. First, disable the `Submit Content` button altogether and remove it from the menu, forcing users to submit files through the changelist window. The second option, which is still a bit of an assumption, is to implement validation directly within **Perforce** itself, bypassing the `EditorValidatorSubsystem`. The third option is to expose both the validation C++ function and the C++ function that the `Submit Content` button connects to, allowing to integrate them into a customized Python widget, replacing the original button with a new button that includes proper validation function connected. 

Well, the first option is quite easy to do, the source code of the UI is in the file `\Engine\Source\Editor\StatusBar\Private\SourceControlMenuHelpers.cpp`

![remove-button](/post-img/unrealtools/ue-validator/remove-button.png){: .shadow}

However, the downside is that users would no longer have a quick submit option. 

As for the second and third options, I’ll need to spend more time investigating their feasibility before implementing them.










