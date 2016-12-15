# STIG
## STIG/Security Control XML Transformations
-------------------------------------------------------------------------------
### Resources:
-------------------------------------------------------------------------------
8500.2 controls compiled from github content
	https://raw.githubusercontent.com/adamcrosby/8500controls/master/8500controls.xml

800-53 controls from nist
	https://nvd.nist.gov/static/feeds/xml/sp80053/rev4/800-53-controls.xml

U_CCI_List comes from the IASE website:
	http://iase.disa.mil/stigs/cci/Pages/index.aspx

controlMapping is generated by tables on the RMF Knowledge Service
	https://rmfks.osd.mil/rmf/General/SecurityControls/Pages/Comparisonof85002and800-53.aspx
	https://rmfks.osd.mil/rmf/_layouts/download.aspx?SourceURL=https://rmfks.osd.mil/rmf/SiteResources/References/Reference%20Library/DoDI%208500.2-%20NIST%20SP%20800-53%20Rev%204%20Crosswalk_12_22_14.xlsx
	
--------------------------------------------------------------------------------------------------------------------------------
### Execution:
--------------------------------------------------------------------------------------------------------------------------------
To regenerate the HTML files with each DISA STIG Release, execute the following in PowerShell:

* Generate the control mapping file based off the 85002 to 800-53 Excel spreadsheet provided by the RMF Knowledge Service.

* From the Excel file, remove first two lines and save as CSV with headers...then execute the following powershell
--------------------------------------------------------------------------------------------------------------------------------

```
clear;

[xml]$cm = New-Object system.Xml.XmlDocument
$cm.LoadXml('<?xml version="1.0" encoding="utf-8" standalone="yes"?><controlMapping><control /></controlMapping>')

$source = import-csv 'DoDI 8500.2- NIST SP 800-53 Rev 4 Crosswalk_12_22_14.csv'
foreach($s in $source){
	$s.'NIST SP 800-53 Rev 4 Security Control Acronym' -split ';' | %{

		$control = $cm.createelement("control") 
		
		$rmf = $cm.createelement("rmf") 
		$rmfText = $cm.CreateTextNode( $($_.trim()) )
		$rmf.AppendChild($rmfText) | out-null
		$control.appendChild($rmf)
		
		$diacap = $cm.createelement("diacap")
		$diacapText = $cm.CreateTextNode( $($s.'DoDI 8500.2 Security Control Acronym').trim() ) 
		$diacap.AppendChild($diacapText) | out-null
		$control.appendChild($diacap)
		
		$cm.controlMapping.AppendChild($control) | out-null
	}
}

$cm.save("$($pwd)\controlMapping.xml");
```

-------------------------------------------------------------------------------
* Generate the Configuration file (lists available STIGS) based off all the stigs available in a single folder.  From the folder with the STIGS, execute
-------------------------------------------------------------------------------

```
clear;
[xml]$c= New-Object system.Xml.XmlDocument
$c.LoadXml('<?xml version="1.0" encoding="utf-8" standalone="yes"?><configurations><menu><item file-ref=""></item></menu></configurations>')

ls | sort -property name | ? { $_.name -like '*xccdf*' } | % {
	$stigXml = ([xml](gc $($_.name)))
	$fileref =  $_.basename;
	$stig_title = $stigXml.Benchmark.title
	$selectTitle = $stig_title -replace 'Security Technical Implementation Guide','' -replace '(STIG)','' -replace 'Benchmark','' -replace 'Security Implementation Guide','' -replace 'Security Requirements Guide','' -replace '\(SRG\)',''
	
	$selectTitle += ' - V' + $stigXml.Benchmark.version
	$selectTitle += 'R' + $( ($stigXml.Benchmark.'plain-text'.'#text' -replace 'Release: ','' ) -split 'Date:' | select -first 1)
	
	$selectTitle = $selectTitle -replace 'Benchmark',''
	
	if($_.name -like '*benchmark*'){
		$selectTitle += ' (Benchmark)'
	}else{
		$selectTitle += ' (STIG)'
	}
	$selectTitle = $selectTitle -replace '\(\)',' '
	$selectTitle = $selectTitle -replace '[ ]+',' '
	
	write-host $selectTitle
	
	$item = $c.createelement("item")
	$item.SetAttribute("file-ref", $fileref) | out-null
	$item.SetAttribute("stig-title", $stig_title) | out-null
	$xmlText = $c.CreateTextNode($selectTitle)
	$item.AppendChild($xmlText) | out-null

	$c.configurations.menu.AppendChild($item) | out-null
}


$item = $c.createelement("item")
$item.SetAttribute("file-ref", '8500controls') | out-null
$item.SetAttribute("stig-title", "8500-2") | out-null
$xmlText = $c.CreateTextNode('8500-2')
$item.AppendChild($xmlText) | out-null
$c.configurations.menu.AppendChild($item) | out-null

$item = $c.createelement("item")
$item.SetAttribute("file-ref", '80053controls') | out-null
$item.SetAttribute("stig-title", "800-53") | out-null
$xmlText = $c.CreateTextNode('800-53')
$item.AppendChild($xmlText) | out-null
$c.configurations.menu.AppendChild($item) | out-null	

$c.save("$($pwd)\configurations.xml");
```

-------------------------------------------------------------------------------
* Transform from xml/xsl to HTML with the following code, after the configurationfile has been updated
-------------------------------------------------------------------------------

```
clear;
function processXslt {
	param(
		[string]$xmlPath, 
		[string] $xslPath,
		$argParms = $null
	)
	
	if($argParms -ne $null){
		$arglist = new-object System.Xml.Xsl.XsltArgumentList
		
		$argParms.keys | % {
			$arglist.AddParam($_, "", $argParms.$_);
		}
		
	}else{
		$arglist = $null;
	}
	
	$xmlContent = [string](gc $xmlPath)
	
	$inputstream = new-object System.IO.MemoryStream
	$xmlvar = new-object System.IO.StreamWriter($inputstream)
	$xmlvar.Write( $xmlContent)
	$xmlvar.Flush()
	$inputstream.position = 0
	$xmlObj = new-object System.Xml.XmlTextReader($inputstream)
	$output = New-Object System.IO.MemoryStream
	$xslt = New-Object System.Xml.Xsl.XslCompiledTransform
	
	$reader = new-object System.IO.StreamReader($output)
	
	$resolver = New-Object System.Xml.XmlUrlResolver
	$xslSettings = New-Object System.Xml.Xsl.XsltSettings($false,$true)
	$xslSettings.EnableDocumentFunction = $true
	$xslt.Load($xslPath,$xslSettings, $resolver)
			
	$xslt.Transform($xmlObj, $arglist, $output)
	$output.position = 0
	$transformed = [string]$reader.ReadToEnd()
	$reader.Close()
	return $transformed
}

ls | ? { $_.name -like '*xccdf*'} | ? { $_.extension -eq '.xml' } | % {write-host $_.basename; processXslt -xmlPath ($_.name) -xslPath "$($pwd)\STIG_unclass.xsl" | set-content "$($_.basename).html";}

processXslt -xmlPath "$($pwd)\80053controls.xml" -xslPath "$($pwd)\80053_controls_unclass.xsl" | set-content "80053controls.html"
processXslt -xmlPath "$($pwd)\8500controls.xml" -xslPath "$($pwd)\8500_controls_unclass.xsl" | set-content "8500controls.html"
```

-------------------------------------------------------------------------------
