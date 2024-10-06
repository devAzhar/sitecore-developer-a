**Scriban - Include Common Functions in Rendering Variants**

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

**Define a structure in Sitecore content tree to store common functions**
We can choose a place in Shared location to store common functions. E.g.

`/sitecore/content/Websites/Shared/Presentation/Rendering Variants/_ScribanBase/CommonScribanScripts`

This can be a rendering variant, we won't use it directly in any rendering but we can just use it to store common functions. 

image-1

You can either define all functions in one item or you can create multiple items under the CommonScribanScripts item.

Current implementation is for global common functions but the same concept can be extended to either do Tenant/Site based common functions or on a lower level like type of functions. 

**Backend code and configuration to use common functions**
```
namespace YourNameSpace.Setup.Pipelines.Scriban
{
  using System.Linq;
  using System.Text;
  using System.Web;
  using Sitecore.Data.Items;
  using Sitecore.XA.Foundation.Scriban.Fields;
  using Sitecore.XA.Foundation.Scriban.Pipelines.RenderVariantField;
  using Sitecore.XA.Foundation.Scriban.Services;
  using Sitecore.XA.Foundation.Variants.Abstractions.Pipelines.RenderVariantField;
  using System.Collections.Generic;

  public static class Constants
  {
    /// <summary>
    /// Scriban related constants
    /// </summary>
    public static class Scriban
    {
      /// <summary>
      /// ID for Common Scriban Scripts
      /// /sitecore/content/Websites/Shared/Presentation/Rendering Variants/_ScribanBase/CommonScribanScripts
      /// </summary>
      public static readonly Sitecore.Data.ID CommonScribanScriptId = new Sitecore.Data.ID("#REPLACE WITH YOUR ITEM ID#");
      public static readonly string ScriptToken = "{{ScribanCommonFunctions}}";
      public static readonly string TemplateFieldName = "Template";
      public static readonly string ItemKey = "GetCommonScribanScriptContent";
      public static readonly string Seprator = "\r\n";
    }
  }
  /// <summary>
  /// Include Common Scriban Scripts And RenderScriban
  /// Extends out of the box implementation of RenderScriban processor
  /// If the scriban script contains the token for common functions, the token will be replaced with the common function scripts from content tree 
  /// </summary>
  public class IncludeCommonScribanScriptsAndRenderScriban : RenderScriban
  {
    /// <summary>
    /// Default Constructor for IncludeCommonScribanScriptsAndRenderScriban
    /// </summary>
    /// <param name="renderer">IScribanTemplateRenderer renderer</param>
    public IncludeCommonScribanScriptsAndRenderScriban(IScribanTemplateRenderer renderer) : base(renderer)
    {
    }

    /// <summary>
    /// Render Field
    /// Add the common functions scriban script to the rendering variant script
    /// Run the actual implementation of RenderField from the base class
    /// </summary>
    /// <param name="args">RenderVariantFieldArgs args</param>
    public override void RenderField(RenderVariantFieldArgs args)
    {
      // Only run this processor for Scriban Variant
      if (!(args.VariantField is VariantScriban variantScriban))
      {
        return;
      }

      // Get the scriban script of the rendering variant
      var scribanTemplate = variantScriban.Template;

      // If the rendering variant scriban script contains the common function token
      if (scribanTemplate.Contains(Constants.Scriban.ScriptToken))
      {
        // Get the combined common scriban script
        var commonScript = GetCommonScribanScriptContent();

        // Replace the token with the common functions script
        scribanTemplate = scribanTemplate.Replace(Constants.Scriban.ScriptToken, commonScript);

        // Update back in the Variant Field in the args 
        variantScriban.Template = scribanTemplate;
        args.VariantField = variantScriban;
      }

      // Call base class RenderField to perform the rest of the processing
      base.RenderField(args);
    }

    /// <summary>
    /// Get Common Scriban Script Content
    /// Only called once per page load - provided the main rendering variant script contains the script include token
    /// </summary>
    /// <returns>Combined scriban scripts from common function Sitecore items</returns>
    private static string GetCommonScribanScriptContent()
    {
      // Make sure to run only once per page request and then cache it for all next set of requests
      // As the common functions will remain the same for all rendering variants,
      // we can reuse the calculated value for all variants used on the page.
      var itemKey = Constants.Scriban.ItemKey;

      // Already calculate for the current page request, return it as is
      if (HttpContext.Current?.Items[itemKey] is string itemKeyValue)
      {
        return itemKeyValue;
      }

      var script = new StringBuilder();

      // Will iterate through all items (if any) and combine the scripts into one string
      var rootItem = Sitecore.Context.Database.GetItem(Constants.Scriban.CommonScribanScriptId);
      var allChildren = rootItem?.GetChildren().ToList();

      if (allChildren != null)
      {
        foreach (var scriban in allChildren)
        {
          var content = scriban[Constants.Scriban.TemplateFieldName];

          if (!string.IsNullOrEmpty(content))
          {
            script = script.Append($"{content}{Constants.Scriban.Seprator}");
          }
        }
      }

      var commonScript = script.ToString();

      // Save for next set of scriban variants on the page
      if (HttpContext.Current != null)
      {
        HttpContext.Current.Items[itemKey] = commonScript;
      }

      return commonScript;
    }
  }
}
```
You also need to provide configuration so that your implementation is used.

```
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <pipelines>
      <renderVariantField>
<!--CHANGE the processor type to match your own class and namespace names-->
        <processor type="YourNameSpace.Setup.Pipelines.Scriban.IncludeCommonScribanScriptsAndRenderScriban, YourNameSpace.Setup" patch:instead="processor[@type='Sitecore.XA.Foundation.Scriban.Pipelines.RenderVariantField.RenderScriban, Sitecore.XA.Foundation.Scriban']" resolve="true" />
      </renderVariantField>
    </pipelines>
  </sitecore>
</configuration>
```

**Using this functionality**
As you can see the processor will only do something if your rendering variant will contain the defined token like {{ScribanCommonFunctions}}.

Once your scriban has this token {{ScribanCommonFunctions}} at the top, the rest of the scriban will have access to all functions defined in the content tree. 
