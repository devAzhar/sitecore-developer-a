#Scriban - Include Common Functions in Rendering Variants

[Scriban](https://github.com/scriban/scriban/blob/master/doc/language.md) is a greate way to develop Rendering Variants in Sitecore SXA solutions.

Out of the box, Scriban in SXA does not provide a way to re-use functions across different scriban based rendering varaints.

sc_evaluate/sc_execute can do the trick for one rendering variant but there is no option to do it across different rendering varaints.

E.g. assume you are using a utility function to read rendering parameters from a drop-down with an option to provide defaults if parameter is not configured in the presentation details.

This is just an example scnario and there can be many more utility functions that you will need in multiple rendering varniants.

```
{{
    func GetRenderingParameterValue
        $fieldName = $0
        $defaultValue = $1
        $retValue = ""
       
        $paramValue = (sc_parameter $fieldName)

        if ($paramValue | string.size) > 0
            $item = sc_getitem $paramValue
           
            if $item
                $retValue = $item.Value.raw
            end
        end
        if ($retValue | string.size) == 0
            $retValue = $defaultValue
        end

        ret $retValue
    end
}}
```

One option is to include this script in all scribans where needed but that causes a maintinance issue, 
if there is a need to change some logic in this function, then you will have to go to all the scribans where this function is defined and make changes.

I have come up with the a solution that can help us to define such utility functions in one place and then re-use these as is where needed.


