# lua condig

```lua
target("ebpf_program1") -- basic
    add_files("src/opensnoop.bpf.c")

target("ebpf_program1")
    set_kind("uprobe")
    attach_to("handler_entry_uprobe", "lua_pcall") -- attach to lua_pcall in uprobe
    on_event(function (event)
        sort(event)
        os.print("uprobe event: ", event)
        stop("ebpf_program1")
    end)
    add_files("src/uprobe.bpf.c")

entry(function (arg)   -- replace the default entry with 
    if arg == "uprobe" then
        run("ebpf_program1")
        sleep(1000)
        run("ebpf_program2")
    else
        run("ebpf_program1")
    end
end)
```