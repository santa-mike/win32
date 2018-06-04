---
title: opclib.cpp
description: opclib.cpp
ms.assetid: 33c3bacc-e932-44ab-ae75-f9326cfd94c4
keywords:
- Packaging APIs,set author sample
- packaging,set author sample
- packages,set author sample
- Packaging APIs,sample applications
- packaging,sample applications
- packages,sample applications
- set author sample,opclib.cpp
- Packaging APIs,opclib.cpp
- packaging,opclib.cpp
- packages,opclib.cpp
- opclib.cpp
- sample applications,set author
ms.technology: desktop
ms.prod: windows
ms.author: windowssdkdev
ms.topic: article
ms.date: 05/31/2018
---

# opclib.cpp


```C++
/*****************************************************************************
*
* File: opclib.cpp
*
* Description:
* This file contains definitions of utility functions that wrap some
* common operations needed when working with most Open Packaging Conventions
* (OPC) conformant files. The OPC specification is ECMA-376 Part 2.
*
* ------------------------------------
*
*  This file is part of the Microsoft Windows SDK Code Samples.
* 
*  Copyright (C) Microsoft Corporation.  All rights reserved.
* 
* This source code is intended only as a supplement to Microsoft
* Development Tools and/or on-line documentation.  See these other
* materials for detailed information regarding Microsoft code samples.
* 
* THIS CODE AND INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY
* KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE
* IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A
* PARTICULAR PURPOSE.
* 
****************************************************************************/

#include "stdio.h"
#include "windows.h"
#include "shlobj.h"
#include "msopc.h"
#include "msxml6.h"

#include "opclib.h"
#include "util.h"

namespace opclib
{
// The definitive way to find a part of interest in a package, is to  use a
// relationship type to find the relationship that targets the part and resolve
// the part name.

// The relationship type (core-properties relationship type) of the
// relationship targeting the Core Properties part, as defined by the OPC
// (ECMA-376 Part 2).
static const WCHAR g_corePropertiesRelationshipType[] = 
    L"http://schemas.openxmlformats.org/package/2006/relationships/metadata/core-properties";

// The expected content type of the Core Properties part, as defined by the OPC
// (ECMA-376 Part 2).
static const WCHAR g_corePropertiesContentType[] = 
    L"application/vnd.openxmlformats-package.core-properties+xml";

// Namespaces used in XPath selection queries for the Core Properties part.
// This includes a number of namespaces used in the Core Properties 
// part as specified by the OPC.
static const WCHAR g_corePropertiesSelectionNamespaces[] =
    L"xmlns:cp='http://schemas.openxmlformats.org/package/2006/metadata/core-properties' "
    L"xmlns:dc='http://purl.org/dc/elements/1.1/' "
    L"xmlns:dcterms='http://purl.org/dc/terms/' "
    L"xmlns:dcmitype='http://purl.org/dc/dcmitype/' "
    L"xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'";


//////////////////////////////////////////////////////////////////////////////
// Description:
// Load a package file into a package object to be read.
//////////////////////////////////////////////////////////////////////////////
HRESULT
LoadPackage(
    IOpcFactory *factory,
    LPCWSTR packageName,
    IOpcPackage **outPackage
    )
{
    IStream * sourceFileStream = NULL;

    // Note: Do not use a writable stream to overwrite the data of a package
    // that is read.

    // Create a read-only stream over the package to prevent errors caused by
    // simultaneously writing and reading from a package.
    HRESULT hr = factory->CreateStreamOnFile(
                    packageName, 
                    OPC_STREAM_IO_READ, 
                    NULL, 
                    0, 
                    &amp;sourceFileStream
                    );

    if (SUCCEEDED(hr))
    {
        // Note: If a part is modified, it is accessed at least twice for
        // reading and writing. Use the OPC_CACHE_ON_ACCESS flag to reduce
        // overhead incurred by accessing a package component (in this case, a
        // part) multiple times.

        // Read the package into a package object.
        // Note: A stream used to read a package is active for the lifetime of
        // the package object into which it is read. 
        hr = factory->ReadPackageFromStream(
                sourceFileStream, 
                OPC_CACHE_ON_ACCESS, 
                outPackage
                );
    }

    // Release resources
    if (sourceFileStream)
    {
        sourceFileStream->Release();
        sourceFileStream = NULL;
    }

    return hr;
}


//////////////////////////////////////////////////////////////////////////////
// Description:
// Save a package with the specified package file name.
//
// Note: Changes made to a package through a package object are not saved until
// the package is written.
//////////////////////////////////////////////////////////////////////////////
HRESULT
SavePackage(
    IOpcFactory *factory,
    IOpcPackage *package,
    LPCWSTR targetFileName
    )
{
    IStream * targetFileStream = NULL;
    // Note: Do not use a writable stream to overwrite the data of a package
    // that is read.

    // Create a writable stream over the specified target file name.
    HRESULT hr = factory->CreateStreamOnFile(
                    targetFileName, 
                    OPC_STREAM_IO_WRITE, 
                    NULL, 
                    0, 
                    &amp;targetFileStream
                    );

    if (SUCCEEDED(hr))
    {
        // After a stream over the specified file is created successfully,
        // write package data to the file.
        hr = factory->WritePackageToStream(
                package, 
                OPC_WRITE_DEFAULT, 
                targetFileStream
                );
    }

    // Release resources
    if (targetFileStream)
    {
        targetFileStream->Release();
        targetFileStream = NULL;
    }

    return hr;
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// If the target of the relationship is a part, get the relative URI of the
// target and resolve this URI to the part name using the URI of the 
// relationship's source as the base URI.
//////////////////////////////////////////////////////////////////////////////
HRESULT
ResolveTargetUriToPart(
    IOpcRelationship *relationship,
    IOpcPartUri **resolvedUri
    )
{
    IOpcUri * sourceUri = NULL;
    IUri * targetUri = NULL;
    OPC_URI_TARGET_MODE targetMode;

    // Get the target mode of the relationship.
    HRESULT hr = relationship->GetTargetMode(&amp;targetMode);
    if (SUCCEEDED(hr) &amp;&amp; targetMode != OPC_URI_TARGET_MODE_INTERNAL)
    {
        // The target mode of the relationship was not internal.
        // Function fails because the relationship does not target a part
        // therefore, no part name can be resolved.
        hr = E_FAIL;
    }
    // Get the segments of the URI and turn it into a valid part URI.
    // The target should be resolved against the source URI of the 
    // relationship to expand relative URIs into part URIs with absolute 
    // paths.
    if (SUCCEEDED(hr))
    {
        // Get a URI for the relationship's target. This URI might be
        // relative to the source of the relationship.
        hr = relationship->GetTargetUri(&amp;targetUri);
    }
    if (SUCCEEDED(hr))
    {
        // Get the URI for the relationship's source.
        hr = relationship->GetSourceUri(&amp;sourceUri);
    }
    if (SUCCEEDED(hr))
    {
        // Form the API representation of the resultant part name.
        hr = sourceUri->CombinePartUri(targetUri, resolvedUri);
    }

    // Release resources
    if (sourceUri)
    {
        sourceUri->Release();
        sourceUri = NULL;
    }

    if (targetUri)
    {
        targetUri->Release();
        targetUri = NULL;
    }

    return hr;
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// Find a part that is the target of a package relationship based on the
// specified relationship type. If a content type is specified, check the
// content type of the part to ensure that only a part with the specified
// is retrieved.
// 
// Note: This function finds the first, arbitrary part that is the target of
// a relationship of the specified type and has the specified content type, if
// a content type is provided.
//
// Example of the relationship markup for a relationship that targets a part:
//   <Relationship Id="rId1" 
//      Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" 
//      Target="word/document.xml" />
//
// The "Type" attribute (relationship type) is the definitive way to find a 
// part of interest in a package. The relationship type is defined by the
// package designer or in the OPC and is therefore consistent and predictable.
// 
// In contrast, the "Id" and "Target" attributes are arbitrary, and not recommended
// for finding specific parts of interest.
//
// Using a relationship type to find a part of interest requires a few steps.
// 1. Identify the relationship type, as defined by the package designer or the
//    OPC, of the relationship(s) that has the part of interest as the target.
// 2. Get relationships of that relationship type from a relationship set.
//    Note: It may be necessary to check that the number of retrieved 
//          relationships conforms to an expected number of relationships
//          specified by the format designer or in the OPC.
// 3. If the target of a retrieved relationship is a part, resolve the part
//    name.
// 4. Get the part.
// 5. If an expected content type is defined by the package designer or the OPC,
//    ensure that the part has the correct content type.
// 6. If the found part has the expected content type, or if no expected
//    content type is defined, return the part.
//////////////////////////////////////////////////////////////////////////////
HRESULT 
FindPartByRelationshipType(
    IOpcPackage *package,
    LPCWSTR relationshipType, // Relationship type used to find the part.
    LPCWSTR contentType, // Expected content type of part (optional).
    IOpcPart **part
    )
{
    *part = NULL; // Enable checks against value of *part.

    IOpcRelationshipSet * packageRels = NULL;
    IOpcRelationshipEnumerator * packageRelsEnum = NULL;
    IOpcPartSet * partSet = NULL;
    BOOL hasNext = false;

    HRESULT hr = package->GetPartSet(&amp;partSet);
    if (SUCCEEDED(hr))
    {
        // Get package relationships stored in the package's Relationships part.
        hr = package->GetRelationshipSet(&amp;packageRels);
    }
    if (SUCCEEDED(hr))
    {
        // Get package relationships of the specified relationship type.
        hr = packageRels->GetEnumeratorForType(
                relationshipType,
                &amp;packageRelsEnum
                );
    }

    // Note: Though not performed by this sample, it may be necessary to check
    // that the number of retrieved relationships conforms to the expected
    // number of relationships specified by the format designer or in the OPC.

    if (SUCCEEDED(hr))
    {
        // Move pointer to first package relationship.
        hr = packageRelsEnum->MoveNext(&amp;hasNext);
    }

    // Find the first, arbitrary part that is the target of a relationship
    // of the specified type and has the specified content type, if a content
    // type is provided. Abandon search when an error is encountered, when
    // there are no more relationships in the enumerator, or when a part is
    // found.
    while (SUCCEEDED(hr) &amp;&amp; hasNext &amp;&amp; *part == NULL)
    {
        IOpcPartUri * partUri = NULL;
        IOpcRelationship * currentRel = NULL;
        BOOL partExists = FALSE;

        hr = packageRelsEnum->GetCurrent(&amp;currentRel);
        if (SUCCEEDED(hr))
        {
            // There was a relationship of the specified type.
            // Try to resolve the part name of the relationship's target.
            hr = ResolveTargetUriToPart(currentRel, &amp;partUri);
        }
        if (SUCCEEDED(hr))
        {
            // Part name resolved. Check that a part with that part name
            // exists in the package.
            hr = partSet->PartExists(partUri, &amp;partExists);
        }
        if (SUCCEEDED(hr) &amp;&amp; partExists)
        {
            // A part with the resolved part name exists in the package, so
            // get a pointer to that part.

            LPWSTR currentContentType = NULL;
            IOpcPart * currentPart = NULL;

            hr = partSet->GetPart(partUri, &amp;currentPart);
            if (SUCCEEDED(hr) &amp;&amp; contentType != NULL)
            {
                // Content type specified.
                // Get the content type of the part.
                hr = currentPart->GetContentType(&amp;currentContentType);

                // Compare the content type of the part with the specified
                // content type.
                if (SUCCEEDED(hr) &amp;&amp;
                    0 == wcscmp(contentType, currentContentType))
                {
                    // Part content type matches specified content type.
                    // Part found.
                    *part = currentPart;
                    currentPart = NULL;
                }
            }
            if (SUCCEEDED(hr) &amp;&amp; contentType == NULL)
            {
                // Content type not specified.
                // Part found.
                *part = currentPart;
                currentPart = NULL;
            }

            // Release resources
            CoTaskMemFree(static_cast<LPVOID>(currentContentType));
            
            if (currentPart)
            {
                currentPart->Release();
                currentPart = NULL;
            }
        }
        // Get the next relationship of the specified type.
        if (SUCCEEDED(hr))
        {
            hr = packageRelsEnum->MoveNext(&amp;hasNext);
        }

        // Release resources
        if (partUri)
        {
            partUri->Release();
            partUri = NULL;
        }

        if (currentRel)
        {
            currentRel->Release();
            currentRel = NULL;
        }
    }

    if (SUCCEEDED(hr) &amp;&amp; *part == NULL)
    {
        // Loop complete without errors and no part found.
        hr = E_FAIL;
    }

    // Release resources
    if (packageRels)
    {
        packageRels->Release();
        packageRels = NULL;
    }

    if (packageRelsEnum)
    {
        packageRelsEnum->Release();
        packageRelsEnum = NULL;
    }

    if (partSet)
    {
        partSet->Release();
        partSet = NULL;
    }

    return hr;
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// Read part content into a new MSXML DOM document. If selection namespaces
// are provided, set up DOM document for XPath queries. Changes made to
// content from the DOM document must be written to the part explicitly.
//////////////////////////////////////////////////////////////////////////////
HRESULT 
DOMFromPart(
    IOpcPart * part,
    LPCWSTR selectionNamespaces,
    IXMLDOMDocument2 **document
    )
{
    IXMLDOMDocument2 * partContentXmlDocument = NULL;
    IStream * partContentStream = NULL;

    HRESULT hr = CoCreateInstance(
                    __uuidof(DOMDocument60), 
                    NULL, 
                    CLSCTX_INPROC_SERVER, 
                    __uuidof(IXMLDOMDocument2), 
                    (LPVOID*)&amp;partContentXmlDocument
                    );

    // If selection namespaces were provided, configure the document 
    // for XPath queries.
    if (SUCCEEDED(hr) &amp;&amp; selectionNamespaces)
    {
        // Note: This sample uses a custom variant wrapper because
        // ATL variants may throw.

        AutoVariant v;

        hr = v.SetBSTRValue(L"XPath");

        if (SUCCEEDED(hr))
        {
            hr = partContentXmlDocument->setProperty(L"SelectionLanguage", v);
        }
        if (SUCCEEDED(hr))
        {
            AutoVariant v;

            hr = v.SetBSTRValue(selectionNamespaces);

            if (SUCCEEDED(hr))
            {
                hr = partContentXmlDocument->setProperty(L"SelectionNamespaces", v);
            }
        }
    }

    // Content stream from a part is read-write, though the DOM tree will only
    // read from it.
    if (SUCCEEDED(hr))
    {
        // Note: The DOM may hold the content stream open for the life of the
        // DOM tree, and the content stream may hold a reference to the package.
        hr = part->GetContentStream(&amp;partContentStream);        
    }

    if (SUCCEEDED(hr))
    {
        VARIANT_BOOL isSuccessful = VARIANT_FALSE;
        AutoVariant vStream;

        vStream.SetObjectValue(partContentStream);
        hr = partContentXmlDocument->load(vStream, &amp;isSuccessful);

        if (SUCCEEDED(hr) &amp;&amp; isSuccessful == VARIANT_FALSE)
        {
            // DOM load failed. This check intentionally simple.
            hr = E_FAIL;
            // For more about DOM load failure, see the MSXML documentation.
        }
    }

    if (SUCCEEDED(hr))
    {
        // DOM loaded from part content
        *document = partContentXmlDocument;
        partContentXmlDocument = NULL;
    }

    // Release resources
    if (partContentXmlDocument)
    {
        partContentXmlDocument->Release();
        partContentXmlDocument = NULL;
    }

    if (partContentStream)
    {
        partContentStream->Release();
        partContentStream = NULL;
    }

    return hr;
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// Helper function that finds the Core Properties part of a package.
//////////////////////////////////////////////////////////////////////////////
HRESULT
FindCorePropertiesPart(
    IOpcPackage *package,
    IOpcPart **part
    )
{
    return FindPartByRelationshipType(
                package,
                g_corePropertiesRelationshipType,
                g_corePropertiesContentType,
                part
                );
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// Write core properties information to the console.
//////////////////////////////////////////////////////////////////////////////
HRESULT
PrintCoreProperties(
    IOpcPackage *package
    )
{
    IOpcPart * corePropertiesPart = NULL;
    IXMLDOMDocument2 * corePropertiesDom = NULL;

    HRESULT hr = FindCorePropertiesPart(
                    package,
                    &amp;corePropertiesPart
                    );

    if (SUCCEEDED(hr))
    {
        hr = DOMFromPart(
                corePropertiesPart, 
                g_corePropertiesSelectionNamespaces, 
                &amp;corePropertiesDom
                );
    }

    // Find & show author name.
    if (SUCCEEDED(hr))
    {
        IXMLDOMNode * creatorNode = NULL;
        BSTR text = NULL;

        hr = corePropertiesDom->selectSingleNode(
                L"//dc:creator",
                &amp;creatorNode
                );

        if (SUCCEEDED(hr) &amp;&amp; creatorNode != NULL)
        {
            hr = creatorNode->get_text(&amp;text);
        }

        if (SUCCEEDED(hr))
        {
            wprintf(L"Author: %s\n", (text != NULL) ? text : L"[missing]");
        }

        // Release resources
        if (creatorNode)
        {
            creatorNode->Release();
            creatorNode = NULL;
        }

        SysFreeString(text);
    }

    // Find & show properties indicating date of last modification and the
    // name of who did it.
    if (SUCCEEDED(hr))
    {
        IXMLDOMNode * modifiedName = NULL;
        IXMLDOMNode * modifiedDate = NULL;
        BSTR nameText = NULL;
        BSTR dateText = NULL;

        hr = corePropertiesDom->selectSingleNode(
                L"//cp:lastModifiedBy",
                &amp;modifiedName
                );

        if (SUCCEEDED(hr) &amp;&amp; modifiedName != NULL)
        {
            hr = modifiedName->get_text(&amp;nameText);
        }

        if (SUCCEEDED(hr))
        {
            hr = corePropertiesDom->selectSingleNode(
                    L"//dcterms:modified",
                    &amp;modifiedDate
                    );
        }

        if (SUCCEEDED(hr) &amp;&amp; modifiedDate != NULL)
        {
            hr = modifiedDate->get_text(&amp;dateText);
        }

        if (SUCCEEDED(hr))
        {
            wprintf(
                L"Last modified by %s on %s\n", 
                (nameText != NULL) ? nameText : L"[missing]", 
                (dateText != NULL) ? dateText : L"[missing]"
                );
        }

        // Release resources
        if (modifiedName)
        {
            modifiedName->Release();
            modifiedName = NULL;
        }

        if (modifiedDate)
        {
            modifiedDate->Release();
            modifiedDate = NULL;
        }

        SysFreeString(nameText);
        SysFreeString(dateText);
    }

    // Release resources
    if (corePropertiesPart)
    {
        corePropertiesPart->Release();
        corePropertiesPart = NULL;
    }

    if (corePropertiesDom)
    {
        corePropertiesDom->Release();
        corePropertiesDom = NULL;
    }

    return hr;
}

//////////////////////////////////////////////////////////////////////////////
// Description:
// Set the modified by and author core properties of a specified package to a
// specified author.
//////////////////////////////////////////////////////////////////////////////
HRESULT
SetAuthorAndModifiedBy(
    IOpcPackage *package,
    LPCWSTR author
    )
{
    IOpcPart * corePropertiesPart = NULL;
    IXMLDOMDocument2 * corePropertiesDom = NULL;
    IXMLDOMNode * creatorNode = NULL;
    IXMLDOMNode * modifiedName = NULL;
    IStream * stream = NULL;
    HRESULT hr = FindCorePropertiesPart(
                    package,
                    &amp;corePropertiesPart
                    );
    if (SUCCEEDED(hr))
    {
        hr = DOMFromPart(
                corePropertiesPart, 
                g_corePropertiesSelectionNamespaces, 
                &amp;corePropertiesDom
                );
    }

    // Change text of the author and modified by properties to the specified
    // author in the DOM.

    if (SUCCEEDED(hr))
    {
        hr = corePropertiesDom->selectSingleNode(
                L"//dc:creator",
                &amp;creatorNode
                );
    }

    if (SUCCEEDED(hr) &amp;&amp; creatorNode != NULL)
    {
        hr = creatorNode->put_text((BSTR)author);
    }

    if (SUCCEEDED(hr))
    {
        hr = corePropertiesDom->selectSingleNode(
                L"//cp:lastModifiedBy",
                &amp;modifiedName
                );
    }

    if (SUCCEEDED(hr) &amp;&amp; modifiedName != NULL)
    {
        hr = modifiedName->put_text((BSTR)author);
    }

    // Note: Changes made using the DOM are not automatically applied to
    // part content.

    // Write the changes to part content of the Core Properties part.
    if (SUCCEEDED(hr))
    {
        // Get the read/write part content stream
        hr = corePropertiesPart->GetContentStream(&amp;stream);
    }

    if (SUCCEEDED(hr))
    {
        // Clear old part content from the stream.
        ULARGE_INTEGER zero = {0};
        hr = stream->SetSize(zero);
    }

    if (SUCCEEDED(hr))
    {
        AutoVariant vStream;

        vStream.SetObjectValue(stream);

        // Write changes made to core properties data to the part content
        // stream of the Core Properties part.
        hr = corePropertiesDom->save(vStream);
    }

    // Release resources
    if (corePropertiesPart)
    {
        corePropertiesPart->Release();
        corePropertiesPart = NULL;
    }

    if (corePropertiesDom)
    {
        corePropertiesDom->Release();
        corePropertiesDom = NULL;
    }

    if (creatorNode)
    {
        creatorNode->Release();
        creatorNode = NULL;
    }

    if (modifiedName)
    {
        modifiedName->Release();
        modifiedName = NULL;
    }

    if (stream)
    {
        stream->Release();
        stream = NULL;
    }

    return hr;
}

} // namespace opclib
```



## Related topics

<dl> <dt>

[Set Author Sample](set-author-sample.md)
</dt> </dl>

 

 



