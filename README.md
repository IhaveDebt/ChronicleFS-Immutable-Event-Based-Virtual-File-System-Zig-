const std = @import("std");

const FileEventType = enum {
    create,
    write,
    delete,
};

const FileEvent = struct {
    path: []const u8,
    event_type: FileEventType,
    content: []const u8,
    timestamp: u64,
};

const FileState = struct {
    exists: bool,
    content: []const u8,
};

const ChronicleFS = struct {
    allocator: std.mem.Allocator,
    events: std.ArrayList(FileEvent),

    pub fn init(allocator: std.mem.Allocator) ChronicleFS {
        return .{
            .allocator = allocator,
            .events = std.ArrayList(FileEvent).init(allocator),
        };
    }

    pub fn append(self: *ChronicleFS, event: FileEvent) !void {
        try self.events.append(event);
    }

    pub fn stateAt(self: *ChronicleFS, path: []const u8, time: u64) FileState {
        var state = FileState{
            .exists = false,
            .content = "",
        };

        for (self.events.items) |e| {
            if (e.timestamp > time) continue;
            if (!std.mem.eql(u8, e.path, path)) continue;

            switch (e.event_type) {
                .create => {
                    state.exists = true;
                    state.content = "";
                },
                .write => {
                    if (state.exists) {
                        state.content = e.content;
                    }
                },
                .delete => {
                    state.exists = false;
                    state.content = "";
                },
            }
        }
        return state;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var fs = ChronicleFS.init(gpa.allocator());

    try fs.append(.{
        .path = "/docs/a.txt",
        .event_type = .create,
        .content = "",
        .timestamp = 1,
    });

    try fs.append(.{
        .path = "/docs/a.txt",
        .event_type = .write,
        .content = "hello",
        .timestamp = 2,
    });

    try fs.append(.{
        .path = "/docs/a.txt",
        .event_type = .write,
        .content = "hello world",
        .timestamp = 3,
    });

    const state = fs.stateAt("/docs/a.txt", 2);
    std.debug.print("Exists: {any}, Content: {s}\n", .{
        state.exists,
        state.content,
    });
}
