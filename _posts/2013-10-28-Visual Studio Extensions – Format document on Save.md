---
layout: post
title: Visual Studio Extensions – Format document on Save
---

The Visual Studio extension [Format Document on Save][1]{:target="_blank"} enables auto formatting of the code when you save a file. Visual Studio supports formatting of the code with the CTRL+E,D or CTRL+E,F key shortcuts but with this extension the command 'Format Document' is executed on Save.

Since Visual Studio 2008 I am using this handy set of tools [Power Commands][2]{:target="_blank"}. One of its features is essential for me: *Format document on save*. From their documentation it is clear what this does. 
>The Format document on save option formats the tabs, spaces, and so on of the document being saved. It is equivalent to pointing to the Edit menu, clicking Advanced, and then clicking Format Document. 
I would also add the key shortcut `CTRL+E,D`

One week ago I switched to Visual Studio 2013 and the extension is not available yet for the new dev environment. Years ago I could do a Macro for this task but Macros are out from Visual Studio. Also addins are deprecated in the new VS.  So I decided to create a simple small extension which does exactly this. I have never created such an extension. But people are doing this so easily and there are so many extensions online. That should be easy. Well… It took me two days (4-5 hours total) for such a simple task.

## Step 1: How should the extension behave?

When the developer presses `CTRL+S` the command `Editor.FormatDocument` should be invoked to format the current document.

## Step 2: How to create a simple Visual Studio Extension?

I started with the docs – [Extending Visual Studio Overview][3]{:target="_blank"}. Soon I found that the only way to hook before the actual document save, format the document and then return to the default chain of commands is possible only with the [IOleCommandTarget][4]{:target="_blank"} interface. In the MSDN there are many examples for different kinds of extensions. There was one for my case: [Walktrough: Using a Shortcut Key with an Editor Extension][5]{:target="_blank"}. I did exactly that tutorial to feel how the things are working. It was a total mess for me. In the end of this step at least I knew that I need *Editor Viewport Adornment project* for my little extension.

## Step 3: Get the sample project working with CTRL+S?

There is no easy way. I just debugged it. Lines 16-17 test for the `CTRL+S` combination.

```c#
internal class KeyBindingCommandFilter : IOleCommandTarget
{
    DTE dte;

    internal bool m_added;

    internal IOleCommandTarget m_nextTarget;

    public KeyBindingCommandFilter(DTE dte)
    {
        this.dte = dte;
    }

    int IOleCommandTarget.Exec(ref Guid pguidCmdGroup, uint nCmdID, uint nCmdexecopt, IntPtr pvaIn, IntPtr pvaOut)
    {
        if (pguidCmdGroup == Guid.Parse(Microsoft.VisualStudio.VSConstants.CMDSETID.StandardCommandSet97_string) && 
            nCmdID == (uint)Microsoft.VisualStudio.VSConstants.VSStd97CmdID.SaveProjectItem)
        {
            dte.ExecuteCommand("Edit.FormatDocument", string.Empty);
        }
        return m_nextTarget.Exec(ref pguidCmdGroup, nCmdID, nCmdexecopt, pvaIn, pvaOut);
    }

    int IOleCommandTarget.QueryStatus(ref Guid pguidCmdGroup, uint cCmds, OLECMD[] prgCmds, IntPtr pCmdText)
    {
        return m_nextTarget.QueryStatus(ref pguidCmdGroup, cCmds, prgCmds, pCmdText);
    }
}
```

## Step 4: How to invoke the command Edit.FormatDocument?

The easiest way to do this is to invoke a command trough the DTE object. The provider from the extensions resolves the DTE instance and passes it to the *KeyBindingCommandFilter*. The rest is piece of cake. Calling the command is in the code sample above. Resolving the DTE and passing it to the filter is below.

```c#
[Export(typeof(IVsTextViewCreationListener))]
[ContentType("text")]
[TextViewRole(PredefinedTextViewRoles.Editable)]
internal class KeyBindingCommandFilterProvider : IVsTextViewCreationListener
{
    [Import(typeof(IVsEditorAdaptersFactoryService))]
    internal IVsEditorAdaptersFactoryService editorFactory = null;

    [Import(typeof(SVsServiceProvider))]
    internal SVsServiceProvider ServiceProvider = null;

    public void VsTextViewCreated(IVsTextView textViewAdapter)
    {
        var dte = (DTE)ServiceProvider.GetService(typeof(DTE));
        AddCommandFilter(textViewAdapter, new KeyBindingCommandFilter(dte));
    }

    void AddCommandFilter(IVsTextView viewAdapter, KeyBindingCommandFilter commandFilter)
    {
        if (commandFilter.m_added == false)
        {
            //get the view adapter from the editor factory
            IOleCommandTarget next;
            int hr = viewAdapter.AddCommandFilter(commandFilter, out next);

            if (hr == VSConstants.S_OK)
            {
                commandFilter.m_added = true;
                //you'll need the next target for Exec and QueryStatus 
                if (next != null)
                    commandFilter.m_nextTarget = next;
            }
        }
    }
}
```

I also added several references. The final list of the references is:

* envdte
* Microsoft.VisualStudio.CoreUtility
* Microsoft.VisualStudio.Editor
* Microsoft.VisualStudio.OLE.Interop
* Microsoft.VisualStudio.Shell.12.0
* Microsoft.VisualStudio.Shell.Immutable.10.0
* Microsoft.VisualStudio.Text.UI
* Microsoft.VisualStudio.TextManager.Interop
* System
* System.ComponentModel.Composition
* System.Core

## Step 5: Submit to the extensions gallery

The final result is uploaded to the gallery [here][1].

The full source code is also available in github [here][6].

------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://marketplace.visualstudio.com/items?itemName=mynkow.FormatdocumentonSave
[2]: https://marketplace.visualstudio.com/items?itemName=VisualStudioProductTeam.PowerCommandsforVisualStudio2010
[3]: https://msdn.microsoft.com/de-de/library/bb165336(v=vs.110).aspx
[4]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms683797(v=vs.85).aspx
[5]: https://msdn.microsoft.com/en-us/library/dd885474(v=vs.100).aspx
[6]: https://github.com/Elders/VSE-FormatDocumentOnSave