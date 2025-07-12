---
layout: post
title:  Merancang Text Editor
date:   2025-07-11 00:05:20 +0700
categories: [programming, c++]
---

Text Editor yang kita perlukan bukan sebuah barang mewah, yang memerlukan waktu lama untuk *loading*, kecuali kalau kita sudah membuat sebuah *workspace* atau *project*. Di sini hanya perlu perintak pendek untuk memanggilnya dari dalam sebuah **Terminal**, semacam `cxx-editor hello.cpp`.

Untuk mengedit sebuah kode program (dalam hal ini adalah kode `C++` sendiri) tentu saja minimal kita memerlukan *syntax highlight* yang benar dan enak dilihat. Kita akan memanfatkan kehadiran `GtkSourceView` versi 5, yang memerlukan `Gtk 4`.

Untuk menyederhanakan urusan, tidak ada tombol-tombol di sini, jadi kita akan memanfaatkan *akselerator* berupa `<Control>s`, `<Control>o`, `<Control>n` dan seterusnya, yang memang sudah umum dipakai.

Kita mulai dari file `main.cpp`:

```cpp
#include "common.h"

using namespace adi;

static void on_start(GtkApplication* app, editor* p) {
    p->startup();
}

static void on_activate(GtkApplication* app, editor* p) {
    p->activate();
}

static void on_open(GtkApplication* app, 
    GFile** files, 
    int nfiles, 
    const char* hints, 
    editor* p) {
    p->open(files, nfiles);
}

int main(int argc, char *argv[]) {
    GtkApplication *app = gtk_application_new("net.adi.editor", 
        G_APPLICATION_HANDLES_OPEN);
    editor* p = new editor(app);
    g_signal_connect(app, "startup", G_CALLBACK (on_start), p);
    g_signal_connect(app, "activate", G_CALLBACK (on_activate), p);
    g_signal_connect(app, "open", G_CALLBACK(on_open), p);
    int status = p->run(argc, argv);
    delete p;
    g_object_unref(app);
    return status;
}
```

File ini hanya mengimpor sebuah `header`, yaitu "common.h", isinya adalah sbb:

```cpp
#pragma once

#include <gtk/gtk.h>
#include <gtksourceview/gtksource.h>
#include <iostream>
#include <string>
#include <vector>

namespace adi {

    struct editor {
        editor(GtkApplication* g);
        ~editor();

        void startup();
        void activate();
        void open(GFile** pp, int nfiles);
        void onWindowDestroyed();
        void saveDocument();
        void showOpenDlg();

        int run(int argc, char** argv);
        void showDocumentContent(const char* path);

    private:
        GtkApplication* app;
        GtkWindow* window;
        GtkScrolledWindow* scroll;
        char* docPath;
        GtkSourceView* view;
        GtkSourceBuffer* sourceBuffer;
        void initView();
    };
}
```

Fungsi-fungsi di dalam `struct adi:editor` ini diimplementasikan di dalam file "editor.cpp", yang akan kita bahas satu persatu di sini. Untuk menyalin semua isinya, [lompat ke bagian berikut](#salin-semua).

Mula-mula, `main()` akan memanggil fungsi objek `adi::editor` yang bernama `run(int argc, char** argv)`. Fungsi tersebut hanya kita implementasikan sbb:

```cpp
int adi::editor::run(int argc, char** argv) {
    return g_application_run(G_APPLICATION(app), argc, argv);
}
```

Setelah aplikasi ini *running*, yang pertama akan dipanggil adalah fungsi `startup()`, yang hanya akan kita pakai untuk menentukan *style* bagi program kita. Untuk ini hanya akan mengurus soal *font* dan ison untuk aplikasi kita.

```cpp
void adi::editor::startup() {
    GdkDisplay* display = gdk_display_get_default ();
    GtkCssProvider* provider = gtk_css_provider_new();

    gtk_style_context_add_provider_for_display(display, 
        GTK_STYLE_PROVIDER(provider), 
        GTK_STYLE_PROVIDER_PRIORITY_APPLICATION);

    const char* css_data = "textview {font: 12pt \"Consolas\";}";

    gtk_css_provider_load_from_string(provider, css_data);
    gtk_window_set_default_icon_name("lambda");
}
```

Setelah itu, program akan memanggil `activate()`, yang akan membuat `Window` untuk aplikasi kita, dan sekaligus menampilkannya.

Ada beberapa hal yang harus dibahas di sini. Jika program dipanggil dengan hanya mengetik namanya, atau `double-click` icon dari dalam `File Manager` yang mendukung ini, maka yang ditampilkan hanya sebuah Window kosong. Untuk memuat file tertentu, kita harus menggunakan kombinasi `Ctrl+o`. Sebaliknya, jika memanggilnya dengan perintah `editor main.cpp`, maka yang akan dipanggil adalah fungsi `open()` berikut:

```cpp
void adi::editor::open(GFile** files, int nfiles) {
    std::cerr << "adi::editor[" << this << "]::open\n"
        << "files: " << files << "\n"
        << "nfiles: " << nfiles << "\n"
        << "Only opening one file, ignore any others...\n";
    activate();
    char* src = g_file_get_path(files[0]);
    showDocumentContent(src);
    g_free(src);
}
```

Perhatikan, untuk *single-view* semacam ini, kita tidak melayani membuka sisa dokumen yang diberikan, dan hanya akan mengabaikannya. Dokumen pertama yang akan ditampilkan isinya.

Fungsi itu mendelegasikan tugas kepada `showDocumentContent()`, yang isinya adalah sbb:

```cpp
void adi::editor::showDocumentContent(const char* path) {
    std::ifstream ifs(path);
    if (!ifs.is_open()) {
        std::cerr << "Could not open file " << path << "\n";
        return;
    }

    std::string content(
        (std::istreambuf_iterator<char>(ifs)),
        std::istreambuf_iterator<char>()
    );

    if (!view) {
        initView();
    }
    gtk_text_buffer_set_text(GTK_TEXT_BUFFER(sourceBuffer), content.c_str(), -1);
    g_free(docPath);
    docPath = g_strdup(path);
}
```

Lagi-lagi, fungsi ini memanggil fungsi *private* yang bernama `initView()`.

```cpp
void adi::editor::initView() {
    if (view) return; // avoid being initialized multiple times

    GtkSourceLanguageManager *lm = gtk_source_language_manager_get_default();
    GtkSourceLanguage* gsl = gtk_source_language_manager_get_language(lm, "cpp");

    sourceBuffer = gtk_source_buffer_new_with_language(gsl);

    GtkWidget* obj = gtk_source_view_new_with_buffer(sourceBuffer);
    view = GTK_SOURCE_VIEW(obj);

    gtk_source_view_set_auto_indent(view, true);
    gtk_source_view_set_enable_snippets(view, true);
    gtk_source_view_set_indent_width(view, 4);
    gtk_source_view_set_show_line_numbers(view, true);

    gtk_text_view_set_wrap_mode(GTK_TEXT_VIEW(view), GTK_WRAP_WORD);

    GtkSourceStyleSchemeManager* ssm = gtk_source_style_scheme_manager_get_default();
    GtkSourceStyleScheme* ss = gtk_source_style_scheme_manager_get_scheme(ssm, "Adwaita-dark");
    gtk_source_buffer_set_style_scheme(sourceBuffer, ss);
    gtk_scrolled_window_set_child(scroll, GTK_WIDGET(view));
}
```


```cpp
void adi::editor::activate() {
    GtkWidget* obj = gtk_application_window_new(app);
    window = GTK_WINDOW(obj);
    gtk_window_set_title(window, "C++ Editor");
    gtk_window_set_default_size(window, 640, 480);

    obj = gtk_scrolled_window_new();
    scroll = GTK_SCROLLED_WINDOW(obj);
    gtk_scrolled_window_set_policy(scroll, GTK_POLICY_AUTOMATIC, GTK_POLICY_AUTOMATIC);
    gtk_window_set_child(window, obj);

    GSimpleAction* save_action = g_simple_action_new("save", nullptr);
    g_signal_connect(save_action, "activate", G_CALLBACK(on_save), this);
    g_action_map_add_action(G_ACTION_MAP(app), G_ACTION(save_action));
    g_object_unref(save_action);

    GSimpleAction* open_action = g_simple_action_new("open", nullptr);
    g_signal_connect(open_action, "activate", G_CALLBACK(on_action_open), this);
    g_action_map_add_action(G_ACTION_MAP(app), G_ACTION(open_action));
    g_object_unref(open_action);

    const char *save_accels[] = { "<Control>s", nullptr };
    gtk_application_set_accels_for_action (app, "app.save", save_accels);

    const char *open_accels[] = { "<Control>o", nullptr };
    gtk_application_set_accels_for_action (app, "app.open", open_accels);

    // Connect the events
    g_signal_connect(window, "destroy", G_CALLBACK(destroy), this);
    gtk_window_present(window);
}
```


## Salin Semua

```cpp
#include "common.h"
#include <fstream>

using namespace adi;

adi::editor::editor(GtkApplication* g)
    :app(g),
    window(nullptr),
    scroll(nullptr),
    view(nullptr),
    sourceBuffer(nullptr) {
    docPath = g_strdup("unnamed.cpp");
}

adi::editor::~editor() {
    window = nullptr;
    app = nullptr;
    g_free(docPath);
}

static void on_save(GSimpleAction* action, GVariant* params, editor* p) {
    p->saveDocument();
}

static void on_action_open(GSimpleAction* action, GVariant* params, editor* p) {
    p->showOpenDlg();
}

static void destroy(GtkWindow window, editor* p) {
    p->onWindowDestroyed();
}

void adi::editor::open(GFile** files, int nfiles) {
    std::cerr << "adi::editor[" << this << "]::open\n"
        << "files: " << files << "\n"
        << "nfiles: " << nfiles << "\n"
        << "Only opening one file, ignore any others...\n";
    activate();
    char* src = g_file_get_path(files[0]);
    showDocumentContent(src);
    g_free(src);
}

static void on_btn_open(GtkWidget* btn, editor* p) {
    p->showOpenDlg();
}

void adi::editor::activate() {
    GtkWidget* obj = gtk_application_window_new(app);
    window = GTK_WINDOW(obj);
    gtk_window_set_title(window, "C++ Editor");
    gtk_window_set_default_size(window, 640, 480);

    obj = gtk_scrolled_window_new();
    scroll = GTK_SCROLLED_WINDOW(obj);
    gtk_scrolled_window_set_policy(scroll, GTK_POLICY_AUTOMATIC, GTK_POLICY_AUTOMATIC);
    gtk_window_set_child(window, obj);

    GSimpleAction* save_action = g_simple_action_new("save", nullptr);
    g_signal_connect(save_action, "activate", G_CALLBACK(on_save), this);
    g_action_map_add_action(G_ACTION_MAP(app), G_ACTION(save_action));
    g_object_unref(save_action);

    GSimpleAction* open_action = g_simple_action_new("open", nullptr);
    g_signal_connect(open_action, "activate", G_CALLBACK(on_action_open), this);
    g_action_map_add_action(G_ACTION_MAP(app), G_ACTION(open_action));
    g_object_unref(open_action);

    const char *save_accels[] = { "<Control>s", nullptr };
    gtk_application_set_accels_for_action (app, "app.save", save_accels);

    const char *open_accels[] = { "<Control>o", nullptr };
    gtk_application_set_accels_for_action (app, "app.open", open_accels);

    // Connect the events
    g_signal_connect(window, "destroy", G_CALLBACK(destroy), this);
    gtk_window_present(window);
}

void adi::editor::onWindowDestroyed() {
    //g_application_quit(G_APPLICATION(app));
    //app = nullptr;
}

void adi::editor::startup() {
    GdkDisplay* display = gdk_display_get_default ();
    GtkCssProvider* provider = gtk_css_provider_new();

    gtk_style_context_add_provider_for_display(display, 
        GTK_STYLE_PROVIDER(provider), 
        GTK_STYLE_PROVIDER_PRIORITY_APPLICATION);

    const char* css_data = "textview {font: 12pt \"Consolas\";}";

    gtk_css_provider_load_from_string(provider, css_data);
    gtk_window_set_default_icon_name("lambda");
}

void adi::editor::saveDocument() {
    GtkTextIter t1, t2;
    GtkTextBuffer* buf = GTK_TEXT_BUFFER(sourceBuffer);
    if (!buf) return;

    gtk_text_buffer_get_start_iter(buf, &t1);
    gtk_text_buffer_get_end_iter(buf, &t2);
    char* psz = gtk_text_buffer_get_text(buf, &t1, &t2, 0);
    FILE* pf = fopen(docPath, "wb");
    if (!pf) {
        std::cerr << "Unable to open " << docPath << " for writing\n";
        g_free(psz);
        return;
    }
    size_t len = fwrite(psz, 1, strlen(psz), pf);
    fclose(pf);
    g_free(psz);
    std::cerr << "Writes " << len << " bytes to " << docPath << "\n";
}

int adi::editor::run(int argc, char** argv) {
    int status = g_application_run(G_APPLICATION (app), argc, argv);
    return status;
}

static void onOpenDialog(GObject* src, GAsyncResult* result, void* user_data) {
    GtkFileDialog* dlg = GTK_FILE_DIALOG(src);
    adi::editor* p = reinterpret_cast<adi::editor*>(user_data);
    GFile* file = gtk_file_dialog_open_finish(dlg, result, nullptr);
    if (file) {
        char* path = g_file_get_path(file);
        p->showDocumentContent(path);
        g_free(path);
        g_object_unref(file);
    }
}

void adi::editor::showOpenDlg() {
    GtkFileDialog* dialog = gtk_file_dialog_new();
    gtk_file_dialog_open(dialog, window, nullptr, onOpenDialog, this);
    g_object_unref(dialog);
}

void adi::editor::showDocumentContent(const char* path) {
    std::ifstream ifs(path);
    if (!ifs.is_open()) {
        std::cerr << "Could not open file " << path << "\n";
        return;
    }

    std::string content(
        (std::istreambuf_iterator<char>(ifs)),
        std::istreambuf_iterator<char>()
    );

    if (!view) {
        initView();
    }
    gtk_text_buffer_set_text(GTK_TEXT_BUFFER(sourceBuffer), content.c_str(), -1);
    g_free(docPath);
    docPath = g_strdup(path);
}

void adi::editor::initView() {
    if (view) return; // avoid being initialized multiple times

    GtkSourceLanguageManager *lm = gtk_source_language_manager_get_default();
    GtkSourceLanguage* gsl = gtk_source_language_manager_get_language(lm, "cpp");

    sourceBuffer = gtk_source_buffer_new_with_language(gsl);

    GtkWidget* obj = gtk_source_view_new_with_buffer(sourceBuffer);
    view = GTK_SOURCE_VIEW(obj);

    gtk_source_view_set_auto_indent(view, true);
    gtk_source_view_set_enable_snippets(view, true);
    gtk_source_view_set_indent_width(view, 4);
    gtk_source_view_set_show_line_numbers(view, true);

    gtk_text_view_set_wrap_mode(GTK_TEXT_VIEW(view), GTK_WRAP_WORD);

    GtkSourceStyleSchemeManager* ssm = gtk_source_style_scheme_manager_get_default();
    GtkSourceStyleScheme* ss = gtk_source_style_scheme_manager_get_scheme(ssm, "Adwaita-dark");
    gtk_source_buffer_set_style_scheme(sourceBuffer, ss);
    gtk_scrolled_window_set_child(scroll, GTK_WIDGET(view));
}
```

